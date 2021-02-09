# etcd 集群运维实践

[TOC]

> etcd 是 Kubernetes 集群的数据核心，最严重的情况是，当 etcd 出问题彻底无法恢复的时候，解决问题的办法可能只有重新搭建一个环境。因此围绕 etcd 相关的运维知识就比较重要。etcd 可以容器化部署，也可以在宿主机自行搭建，以下内容是通用的。

## 集群的备份和恢复

### 添加备份

```shell
#!/bin/bash
IP=123.123.123.123
BACKUP_DIR=/alauda/etcd_bak/
mkdir -p $BACKUP_DIR
export ETCDCTL_API=3
etcdctl --endpoints=http://$IP:2379 snapshot save $BACKUP/snap-$(date +%Y%m%d%H%M).db

# 备份一个节点的数据就可以恢复，实践中，为了防止定时任务配置的节点异常没有生成备份，建议多加几个
```

### 恢复集群

```shell
#!/bin/bash

# 使用 etcdctl snapshot restore 生成各个节点的数据

# 比较关键的变量是
# --data-dir 需要是实际 etcd 运行时的数据目录
# --name  --initial-advertise-peer-urls  需要用各个节点的配置
# --initial-cluster  initial-cluster-token 需要和原集群一致

ETCD_1=10.1.0.5
ETCD_2=10.1.0.6
ETCD_3=10.1.0.7

for i in ETCD_1 ETCD_2 ETCD_3
do

export ETCDCTL_API=3
etcdctl snapshot restore snapshot.db \
--data-dir=/var/lib/etcd \
--name $i \
--initial-cluster ${ETCD_1}=http://${ETCD_1}:2380,${ETCD_2}=http://${ETCD_2}:2380,${ETCD_3}=http://${ETCD_3}:2380 \
--initial-cluster-token k8s_etcd_token \
--initial-advertise-peer-urls http://$i:2380 && \
mv /var/lib/etcd/ etcd_$i

done
```

### 用etcd自动创建的snapdb恢复

```shell
#!/bin/bash 
export ETCDCTL_API=3
etcdctl snapshot restore snapshot.db \
--skip-hash-check \
--data-dir=/var/lib/etcd \
--name 10.1.0.5 \
--initial-cluster 10.1.0.5=http://10.1.0.5:2380,10.1.0.6=http://10.1.0.6:2380,10.1.0.7=http://10.1.0.7:2380 \
--initial-cluster-token k8s_etcd_token \
--initial-advertise-peer-urls http://10.1.0.5:2380

# 和上一条命令唯一的差别是多了  --skip-hash-check  需要跳过完整性校验
# 这种方式不能确保 100% 可恢复，建议还是自己加备份
# 通常恢复后需要做一下数据压缩和碎片整理，可参考相应章节
```

### 踩过的坑

[ 3.0.14版 etcd restore 功能不可用 ] https://github.com/etcd-io/etcd/issues/7533

使用更新的 etcd 即可。

## 集群的扩容

### 从1到3

1. 执行添加

   ```shell
   #!/bin/bash
   export ETCDCTL_API=2
   etcdctl --endpoints=http://10.1.0.6:2379 member add 10.1.0.6 http://10.1.0.6:2380
   etcdctl --endpoints=http://10.1.0.7:2379 member add 10.1.0.7 http://10.1.0.7:2380
   
   # ETCD_NAME="etcd_10.1.0.6" 
   # ETCD_INITIAL_CLUSTER="10.1.0.6=http://10.1.0.6:2380,10.1.0.5=http://10.1.0.5:2380"
   # ETCD_INITIAL_CLUSTER_STATE="existing"
   ```

2. 准备添加的节点 etcd 参数配置

   ```shell
   #!/bin/bash
   /usr/local/bin/etcd 
   --data-dir=/data.etcd 
   --name 10.1.0.6
   --initial-advertise-peer-urls http://10.1.0.6:2380 
   --listen-peer-urls http://10.1.0.6:2380 
   --advertise-client-urls http://10.1.0.6:2379 
   --listen-client-urls http://10.1.0.6:2379 
   --initial-cluster 10.1.0.6=http://10.1.0.6:2380,10.1.0.5=http://10.1.0.5:2380
   --initial-cluster-state exsiting
   --initial-cluster-token k8s_etcd_token
   
   # --initial-cluster 集群所有节点的 name=ip:peer_url
   # --initial-cluster-state exsiting 告诉 etcd 自己归属一个已存在的集群，不要自立门户
   ```

3. 踩过的坑

   从 1 到 3 期间，会经过集群是两节点的状态，这时候可能集群的表现就像挂了，endpoint status 这些命令都不能用，所以我们需要用member add先把集群扩到三节点，然后再依次启动etcd实例，这样做就能确保 etcd 就是健康的。

4. 从3到更多，其实还是member add啦，就放心搞吧。

## 集群加证书


### 生成证书

```shell
curl -s -L -o /usr/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -s -L -o /usr/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x /usr/bin/{cfssl,cfssljson}
cd /etc/kubernetes/pki/etcd
```

```json
#  cat ca-config.json
{
  "signing": {
    "default": {
      "expiry": "100000h"
    },
    "profiles": {
      "server": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "100000h"
      },
      "client": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "100000h"
      }
    }
  }
}
```

```json
#  cat ca-csr.json
{
  "CN": "etcd",
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "O": "Alauda",
      "OU": "PaaS",
      "ST": "Beijing"
    }
  ]
}
```

```json
#  cat server-csr.json
{
  "CN": "etcd-server",
  "hosts": [
    "localhost",
    "0.0.0.0",
    "127.0.0.1",
    "所有master 节点ip ",
    "所有master 节点ip ",
    "所有master 节点ip "
  ],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "O": "Alauda",
      "OU": "PaaS",
      "ST": "Beijing"
    }
  ]
}
```

```json
# cat client-csr.json

{
  "CN": "etcd-client",
  "hosts": [
    ""
  ],
  "key": {
    "algo": "rsa",
    "size": 4096
  },
  "names": [
    {
      "C": "CN",
      "L": "Beijing",
      "O": "Alauda",
      "OU": "PaaS",
      "ST": "Beijing"
    }
  ]
}

```

```bash
cd /etc/kubernetes/pki/etcd

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=server server-csr.json | cfssljson -bare server

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=client client-csr.json | cfssljson -bare client
```

### 首先更新节点的peer-urls

```shell
export ETCDCTL_API=3
etcdctl --endpoints=http://x.x.x.x:2379 member list
   #  1111111111  ..........
   #  2222222222  ..........
   #  3333333333  ..........
etcdctl --endpoints=http://172.30.0.123:2379 member update 1111111111 --peer-urls=https://x.x.x.x:2380
   # 执行三次把三个节点的peer-urls都改成https
```

### 修改配置

```shell
#  vim /etc/kubernetes/main*/etcd.yaml

#  etcd启动命令部分修改 http 为 https，启动状态改成 existing
    - --advertise-client-urls=https://x.x.x.x:2379
    - --initial-advertise-peer-urls=https://x.x.x.x:2380
    - --initial-cluster=xxx=https://x.x.x.x:2380,xxx=https://x.x.x.x:2380,xxx=https://x.x.x.x:2380
    - --listen-client-urls=https://x.x.x.x:2379
    - --listen-peer-urls=https://x.x.x.x:2380
    - --initial-cluster-state=existing

#  etcd 启动命令部分插入
	- --cert-file=/etc/kubernetes/pki/etcd/server.pem
	- --key-file=/etc/kubernetes/pki/etcd/server-key.pem
	- --peer-cert-file=/etc/kubernetes/pki/etcd/server.pem
	- --peer-key-file=/etc/kubernetes/pki/etcd/server-key.pem
	- --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem
	- --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem
	- --peer-client-cert-auth=true
	- --client-cert-auth=true

#  检索hostPath在其后插入
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs

#  检索mountPath在其后插入
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs

```

```shell
#  vim /etc/kubernetes/main*/kube-apiserver.yaml
#  apiserver 启动部分插入，修改 http 为https
	- --etcd-cafile=/etc/kubernetes/pki/etcd/ca.pem
	- --etcd-certfile=/etc/kubernetes/pki/etcd/client.pem
	- --etcd-keyfile=/etc/kubernetes/pki/etcd/client-key.pem
	- --etcd-servers=https://x.x.x.x:2379,https://x.x.x.x:2379,https://x.x.x.x:2379
```

### 遇到的坑

[ etcd加证书后，apiserver的健康检查还是http请求，etcd会一直刷日志 ]

https://github.com/etcd-io/etcd/issues/9285

```verilog
2018-02-06 12:41:06.905234 I | embed: rejected connection from "127.0.0.1:35574" (error "EOF", ServerName "")
```

解决办法：直接去掉apiserver的健康检查，或者把默认的检查命令换成curl (apiserver的镜像里应该没有curl，如果是刚需的话自己重新build一下吧 )

## 集群升级

已经是 v3 的的集群不需要太多的配置，保留数据目录，替换镜像(或者二进制)即可；

v2到v3的升级需要一个merge的操作，我并没有实际的实践过，也不太推荐这样做。

## 集群状态检查

其实上述所有步骤都需要这些命令的辅助——

```shell
#!/bin/bash
# 如果证书的话，去掉--cert --key --cacert 即可
# --endpoints= 需要写了几个节点的url，endpoint status就输出几条信息

export ETCDCTL_API=3

etcdctl \
--endpoints=https://x.x.x.x:2379 \ 
--cert=/etc/kubernetes/pki/etcd/client.pem \
--key=/etc/kubernetes/pki/etcd/client-key.pem \
--cacert=/etc/kubernetes/pki/etcd/ca.pem \
endpoint status -w table

etcdctl --endpoints=xxxx endpoint health

etcdctl --endpoints=xxxx member list

kubectl get cs
```

## 数据操作（删除、压缩、碎片整理）

### 删除

```shell
ETCDCTL_API=2 etcdctl rm --recursive            # v2 的 api 可以这样删除一个“目录”
ETCDCTL_API=3 etcdctl --endpoints=xxx del /xxxxx --prefix # v3 的版本

# 带证书的话，参考上一条添加 --cert --key --cacert 即可
```

遇到的坑：在一个客户环境里发现Kubernetes集群里的 “事件” 超级多，就是 kubectl describe xxx 看到的events部分信息，数据太大导致 etcd 跑的很累，我们就用这样的方式删掉没用的这些数据。

### 碎片整理

```shell
ETCDCTL_API=3 etcdctl --endpoints=xx:xx,xx:xx,xx:xx defrag
ETCDCTL_API=3 etcdctl --endpoints=xx:xx,xx:xx,xx:xx endpoint status # 看数据量
```

### 压缩

```shell
ETCDCTL_API=3 etcdctl --endpoints=xx:xx,xx:xx,xx:xx compact

# 这个在只有 K8s 用的 etcd 集群里作用不太大，可能具体场景我没遇到
# 可参考这个文档
# https://www.cnblogs.com/davygeek/p/8524477.html
# 不过跑一下不碍事

etcd --auto-compaction-retention=1

# 添加这个参数让 etcd 运行时自己去做压缩
```

## 常见问题

1. etcd 对时间很依赖，所以集群里的节点时间一定要同步！
2. 磁盘空间不足，如果磁盘是被 etcd 自己吃完了，就需要考虑压缩和删数据啦
3. 加证书后所有请求就都要带证书了，要不会提示 `context deadline exceeded`
4. 做各个操作时 etcd 启动参数里标明节点状态的要小心，否则需要重新做一遍前面的步骤很麻烦

## 日志收集

etcd 的日志暂时只支持 syslog 和 stdout 两种——  https://github.com/etcd-io/etcd/issues/7936

etcd 的日志在排查故障时很有用，如果我们用宿主机来部署 etcd，日志可以通过 systemd 检索到，但kubeadm 方式启动的 etcd 在容器重启后就会丢失所有历史。我们可以用以下的方案来做——

1. shell 的重定向

   ```shell
   etcd --xxxx --xxxx   >  /var/log/etcd.log 
   # 配合 logratate 来做日志切割
   # 将日志通过 volume 挂载到宿主机
   ```

2. supervisor

   supervisor 从容器刚开始流行时，就是保持服务持续运行很有效的工具

3. sidecar 容器（后续我在github上补充一个例子）

   sidecar可以简单理解为一个pod里有多个容器（比如kubedns）他们彼此可以看到对方的进程，因此我们可以用传统的 strace 来捕捉 etcd进程的输出，然后在sidecar这个容器里和shell重定向一样操作。

   ```
   strace  -e trace=write -s 200 -f -p 1
   ```

## kubeadm 1.13部署的集群

最近我们测试kubernetes 1.13集群时发现了一些有趣的改变，诈一看我们上面的命令就没法用了——

https://kubernetes.io/docs/setup/independent/ha-topology/

区分了 `Stacked etcd topology` 和 `External etcd topology` ，官方的链接了这个图很形象——

![Stacked etcd topology](https://d33wubrfki0l68.cloudfront.net/d1411cded83856552f37911eb4522d9887ca4e83/b94b2/images/kubeadm/kubeadm-ha-topology-stacked-etcd.svg)

这种模式下的 etcd 集群，最明显的差别是容器内 etcd 的initial-cluster 启动参数只有自己的 ip，会有点懵挂了我这该怎么去恢复。其实基本原理没有变，kubeadm 藏了个 configmap，启动参数被放在了这里 ——

```shell
kubectl get cm  etcdcfg -n kube-system -o yaml
```

```yaml
    etcd:
      local:
        serverCertSANs:
        - "192.168.8.21"
        peerCertSANs:
        - "192.168.8.21"
        extraArgs:
          initial-cluster: 192.168.8.21=https://192.168.8.21:2380,192.168.8.22=https://192.168.8.22:2380,192.168.8.20=https://192.168.8.20:2380
          initial-cluster-state: new
          name: 192.168.8.21
          listen-peer-urls: https://192.168.8.21:2380
          listen-client-urls: https://192.168.8.21:2379
          advertise-client-urls: https://192.168.8.21:2379
          initial-advertise-peer-urls: https://192.168.8.21:2380
```

