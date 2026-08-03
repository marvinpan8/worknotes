# k0s 集群部署-Ubantu-24.04

系统部署要求  https://docs.k0sproject.io/v1.35.2+k0s.0/system-requirements/

k0s 配置说明  https://docs.k0sproject.io/v1.35.2+k0s.0/configuration/

k0sctl 配置说明 https://github.com/k0sproject/k0sctl?tab=readme-ov-file#configuration-file

## 版本说明

| 组件       | 版本号           |      | 组件           | 版本号       |
| ---------- | ---------------- | ---- | -------------- | ------------ |
| **ubantu** | **24.04** |      | dashboard      | 1.10.1       |
| **k0s** | **v1.35.2+k0s.0** |      | metrics-server | 0.3.1        |
| **k8s**     | **v1.33.2+k0s** |      | **containerd** | **1.6.33** |
| **etcd**   | **3.5.21**       |      | **ctr** | **1.6.33** |
| **calico** | **v3.31.4-1** |      | **crictl** | **1.27.0** |
| **calicoctl** |                       |      | coredns | 1.14.2-0 |
| **bird** | **v0.3.3+birdv1.6.8** |      |  |  |

## 端口说明
| 协议 | 端口 | 服务 | 用途 | 说明 |
| ---------- | ---------------- | ---- | -------------- | ------------ |
| TCP | **2380** | etcd | controller ⟷ controller | |
| TCP | **6443** | kube-apiserver | worker, CLI ⟶ controller | Authenticated Kubernetes API using mTLS, ServiceAccount tokens with RBAC |
| TCP | 179 | kube-router |	worker ⟷ worker | BGP routing sessions between peers |
| UDP | 4789 |  calico | worker ⟷ worker | Calico VXLAN overlay |
| TCP | 10250 | kubelet | controller, worker ⟶ host * | Authenticated kubelet API for the controller node kube-apiserver (and metrics-server add-ons) using mTLS |
| TCP | **9443** | k0s api | controller ⟷ controller | k0s controller join API, TLS with token auth |
| TCP | 8132 | konnectivity | worker ⟷ controller | Konnectivity is used as "reverse" tunnel between kube-apiserver and worker kubelets |
| TCP | 112 | keepalived | controller ⟷ controller | Only required for control plane load balancing VRRPInstances. Unless unicast is explicitly enabled, port 122 works on the ip address 224.0.0.18. 224.0.0.18 is a multicast IP address defined in RFC 3768. |

## 主机名

设置永久主机名称，然后重新登录:

```properties
# 将 m7-autocv-gpu01 替换为当前主机名
sudo hostnamectl set-hostname app101
cat /etc/hostname
```

*   设置的主机名保存在 `/etc/hostname` 文件中；

- sudo vim /etc/hosts

```properties
10.10.10.101 app101
10.10.10.102 app102
10.10.10.103 app103
10.10.10.104 app104
10.10.10.105 app105
```

### 所有节点修改时区

```properties
sudo timedatectl set-timezone Africa/Conakry
sudo timedatectl show --property=Timezone --value | sudo tee /etc/timezone
# 验证
timedatectl
cat /etc/timezone
date
```



# 下载k0s(所有节点)

```properties
cd /k0s/bin
# 下载k0s文件
wget https://k0sproject/k0s/releases/download/v1.35.2/k0s.0/k0s-v1.35.2+k0s.0-amd64
# 授执行权限
chmod +x k0s-v1.35.2+k0s.0-amd64
# 注意不能放在/usr/bin目录，不然后续会报错
sudo cp k0s-v1.35.2+k0s.0-amd64 /usr/local/bin/k0s
# 查看版本
k0s version
sudo k0s sysinfo
```

## bash 补全命令(所有节点)

```properties
mkdir ~/.bash_completion.d
k0s completion bash > ~/.bash_completion.d/k0s

vim ~/.bashrc
#---最后添加如下代码------------------------------
for compFile in ~/.bash_completion.d/*; do
  [ ! -f "$compFile" ] || source -- "$compFile"
done
unset compFile
#---最后添加如下代码------------------------------
source ~/.profile
# k0s 按 tab 键补全 测试 
```



# 导入镜像

```properties
k0s airgap list-images
# -----翻墙下载导入------------------------------------------------
quay.io/k0sproject/calico-cni:v3.31.4-1
quay.io/k0sproject/calico-kube-controllers:v3.31.4-1
quay.io/k0sproject/calico-node:v3.31.4-1
quay.io/k0sproject/coredns:1.14.2-0
quay.io/k0sproject/apiserver-network-proxy-agent:v0.34.0-1
quay.io/k0sproject/kube-proxy:v1.35.2
quay.io/k0sproject/kube-router:v2.7.1-iptables1.8.11-0
quay.io/k0sproject/cni-node:1.8.0-k0s.0
quay.io/k0sproject/metrics-server:v0.8.1-0
quay.io/k0sproject/pause:3.10.1
```



---

# 【101节点】生成配置文件

```properties
cd /k0s && sudo k0s config create > k0s.yaml
```
### vim /k0s/k0sctl.yaml

```properties
noTaints: true
useExistingK0s: true
dataDir: /data/k0s
kubeletRootDir: /data/kubelet
# suns 会自动加上如下
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.svc." + c.ClusterSpec.Network.ClusterDomain,
    "localhost",
    "127.0.0.1",
```
```properties
apiVersion: k0s.k0sproject.io/v1beta1
kind: ClusterConfig
metadata:
  name: k0s
  namespace: kube-system
spec:
  api:
    address: 10.10.10.101
    onlyBindToAddress: true
    k0sApiPort: 9443
    port: 6443
    sans:
      - 10.10.10.101
      - 10.10.10.102
      - 10.10.10.103
      - 10.10.10.99
    ca:
      certificatesExpireAfter: 26280h
      expiresAfter: 87600h
  network:
    dualStack:
      enabled: false
    provider: calico
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.64.0.0/12
    nodeLocalLoadBalancing:
      enabled: true
      type: EnvoyProxy
      envoyProxy:
        imagePullPolicy: IfNotPresent
        image:
          image: quay.io/k0sproject/envoy-distroless
          version: v1.37.1
    calico:
      mode: bird
      overlay: Never
      ipAutodetectionMethod: kubernetes-internal-ip
    kubeProxy:
      ipvs:
        syncPeriod: 30s
        minSyncPeriod: 0s
        tcpFinTimeout: 0s
        tcpTimeout: 0s
        udpTimeout: 0s
      metricsBindAddress: 10.10.10.101:10249
      mode: ipvs
  storage:
    type: etcd
    etcd:
      externalCluster: 
        endpoints:
          - https://10.10.20.201:2379
          - https://10.10.20.202:2379
          - https://10.10.20.203:2379
        etcdPrefix: k0s-gin-1
        caFile: /k0s/etcd/ssl/ca.pem
        clientCertFile: /k0s/etcd/ssl/client-cert.pem
        clientKeyFile: /k0s/etcd/ssl/client-key.pem
  images:
    repository: 10.10.10.102:80
    default_pull_policy: IfNotPresent
    konnectivity:
      image: quay.io/k0sproject/apiserver-network-proxy-agent
      version: v0.34.0-1       
    metricsserver:
      image: quay.io/k0sproject/metrics-server
      version: v0.8.1-0
    coredns:
      image: quay.io/k0sproject/coredns
      version: 1.14.2-0
    pause:
      image: quay.io/k0sproject/pause
      version: 3.10.1
    kubeproxy:
      image: quay.io/k0sproject/kube-proxy
      version: v1.35.2
    kuberouter:
      cni:
        image: quay.io/k0sproject/kube-router
        version: v2.7.1-iptables1.8.11-0
      cniInstaller:
        image: quay.io/k0sproject/cni-node
        version: 1.8.0-k0s.0
    calico:
      kubecontrollers:
        image: quay.io/k0sproject/calico-kube-controllers
        version: v3.31.4-1
      cni:
        image: quay.io/k0sproject/calico-cni
        version: v3.31.4-1
      node:
        image: quay.io/k0sproject/calico-node
        version: v3.31.4-1
```

- **ETCD配置：**https://docs.k0sproject.io/v1.35.2+k0s.0/configuration/#specstorageetcdexternalcluster

- **k0s的calico配置：** https://docs.k0sproject.io/v1.35.2+k0s.0/configuration/#specnetworkcalico

- **calico官网环境变量配置：** https://docs.tigera.io/calico/latest/reference/configure-calico-node#environment-variables

- **calico mode = bird**，不是 **vxlan** 模式，也不是 **ipip** 模式（deprecated）

- **自动生成的SSL文件路径：/var/lib/k0s/pki**  或自定义的 /data/k0s/pki

- **SAN带IP列表：k0s-api.crt、server.crt、etcd/peer.crt(只带本机IP)**

  ****

### 【101节点】配置ssh免密登录其他节点

```properties
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa
sshpass -p 'ekemp' ssh-copy-id -o StrictHostKeyChecking=no ekemp@10.10.10.101
sshpass -p 'ekemp' ssh-copy-id -o StrictHostKeyChecking=no ekemp@10.10.10.102
sshpass -p 'ekemp' ssh-copy-id -o StrictHostKeyChecking=no ekemp@10.10.10.103
sshpass -p 'ekemp' ssh-copy-id -o StrictHostKeyChecking=no ekemp@10.10.10.104
sshpass -p 'ekemp' ssh-copy-id -o StrictHostKeyChecking=no ekemp@10.10.10.105
cat ~/.ssh/known_hosts
```

## 每台机器 ekemp 用户提权

```properties
sudo visudo
#-------添加如下行---------------------
%sudo   ALL=(ALL) NOPASSWD: ALL
#------------------------------------
按 Ctrl + O（保存）
按 Enter 确认文件名
按 Ctrl + X（退出）
```



---

# ■■■ 手动部署安装

官网步骤：https://docs.k0sproject.io/v1.35.2+k0s.0/k0s-multi-node/#install-k0s

```properties
sudo cp /k0s/bin/k0s-v1.35.2+k0s.0-amd64 /usr/local/bin/k0s
```



## 【101】

- **--no-taints=true  仅首次 start 有效**
- **--enable-worker=true  控制+工作节点**
- **--force 重新安装**
- **--start 立即启动**

```properties
IP_ADDRESS="10.10.10.101"
# 生成 systemd 文件 --force 重新安装
sudo k0s install controller --data-dir=/data/k0s --kubelet-root-dir=/data/kubelet --kubelet-extra-args="--address=${IP_ADDRESS}" --enable-worker=true --no-taints=true -c /k0s/k0s.yaml --start
```

###  【重要】检查组网再启动(不包括自己ip)
```properties
sudo k0s status
# 等待所有 pod agent 是否重启为 Running
sudo k0s kubectl -n kube-system get po -owide
# 等待节点 Ready
sudo k0s kubectl get node -o wide
# 检查 calico 组网，不包括自己ip
sudo k0s kubectl -n kube-system exec calico-node-cbpfh -- calico-node -show-status
```

###  修改 calico mode=bird

```properties
# 修改 ds CALICO_IPV4POOL_IPIP = Never
sudo k0s kubectl -n kube-system edit ds calico-node
# 修改 Calico 配置ipipMode: Never,vxlanMode: Never
sudo k0s kubectl edit ippool
# 重启 calico-node
sudo k0s kubectl -n kube-system rollout restart ds calico-node
# 等待全部 calico pod 重启
sudo k0s kubectl -n kube-system get po -owide | grep calico
# 查看 calico 的 bird 网络模式，不会出现 tunl0 或 vxlan.calico 设备
route -n
# ipvs 方式
sudo ipvsadm -Ln
# 查看 Calico 配置
sudo k0s kubectl get ds -n kube-system
```

## 修改配置重启

```properties
# 1. 验证配置文件语法
sudo k0s config validate -c /k0s/k0s.yaml
# 2. 重启服务使配置生效
sudo k0s stop && sudo k0s start
# 3. 检查服务状态
sudo k0s status
```

## 配置 BGP 到H3C 交换机（参见另一篇）

### 创建共用 join token

```properties
cd /k0s && rm -f token-file-*
# 创建 controller join 24 小时 token, 其他节点也可以用
sudo k0s token create --role=controller --expiry=24h > token-file-c
# 创建 worker join 24 小时 token, 其他节点也可以用
sudo k0s token create --role=worker --expiry=24h > token-file-w

# 拷贝 token
scp token-file-c 10.10.10.102:/k0s/
scp token-file-c 10.10.10.103:/k0s/
scp token-file-w 10.10.10.104:/k0s/
scp token-file-w 10.10.10.105:/k0s/
# 查看 token
sudo k0s kubectl -n kube-system get secret
# 删除 token（跳过）
sudo k0s kubectl -n kube-system delete secret bootstrap-token-sn70i0
```



## 【102~103】 间隔2分钟【重要】

```properties
IP_ADDRESS="10.10.10.102"
# 生成 systemd 文件 --force 重新安装
sudo k0s install controller --data-dir=/data/k0s --kubelet-root-dir=/data/kubelet --kubelet-extra-args="--address=${IP_ADDRESS}" --enable-worker=true --no-taints=true -c /k0s/k0s.yaml --start
```
##  【重要】检查组网再启动(不包括自己ip)
```properties
sudo k0s status
# 等待所有 pod agent 是否重启为 Running
sudo k0s kubectl -n kube-system get po -owide
# 等待节点 Ready
sudo k0s kubectl get node -o wide
# 检查 calico 组网，不包括自己ip
sudo k0s kubectl -n kube-system exec calico-node-cbpfh -- calico-node -show-status
```

## 【104~105】间隔2分钟【重要】

```properties
IP_ADDRESS="10.10.10.104"
# 生成 systemd 文件 --force 重新安装
sudo k0s install worker --data-dir=/data/k0s --kubelet-root-dir=/data/kubelet --kubelet-extra-args="--node-ip=${IP_ADDRESS} --address=${IP_ADDRESS}" --token-file=/k0s/token-file-w --start --labels="node.k0sproject.io/role=worker"
```

### 验证检查

```properties
# 查看 k0s 集群
ps -ef |grep k0s
# 查看 k0s 状态
sudo k0s status 
# 查看节点 所有 k0s 配置
sudo k0s status -o yaml
# 查看日志 controller 节点
journalctl -u k0scontroller -n 1000
# 查看日志 worker 节点
journalctl -u k0sworker -f 
# 查看 k0s 端口 9443 44227
sudo netstat -tunlp |grep k0s
# 查看 k8s 端口 6443 10248 10249 10250 10256 10257 10259
sudo netstat -tunlp |grep kube
# 查看 k8s 集群
sudo k0s kubectl cluster-info
sudo k0s kubectl get node -owide --show-labels
# 检查 控制节点 Taints 为空
sudo k0s kubectl describe node app101 | grep Taints
sudo k0s kubectl -n kube-system get po
# 检查 Kubernetes Service
sudo k0s kubectl -n default get svc kubernetes
# 检查 Endpoints
sudo k0s kubectl -n default get endpoints kubernetes
# 查看详细信息
sudo k0s kubectl -n default describe svc kubernetes
# 查看 calico 的 bird 网络模式，不会出现 tunl0 或 vxlan.calico 设备
route -n
# ipvs 方式
sudo ipvsadm -Ln
```
## 每三年证书轮换

```properties
# 查看 自动生成 证书的有效期
sudo find /data/k0s/pki/ -type f -name "*.crt" -print0 |   xargs -0 -I {} sh -c 'echo "=== {} ===" && openssl x509 -noout -dates -in {}'
# 证书 过期 或 删除 重启自动轮换
sudo systemctl restart k0scontroller
sudo systemctl status k0scontroller
```



---

# ■■■ 手动删除服务

### 1. 清理 containerd 中的 k8s 数据
```properties
cc image list
cc ps -a

sudo ctr --address=/run/k0s/containerd.sock -n k8s.io container delete
sudo ctr --address=/run/k0s/containerd.sock -n k8s.io image ls
sudo ctr --address=/run/k0s/containerd.sock -n k8s.io image delete --all
```
### 2. 删除节点

```properties
# 清除已安装的系统服务、数据目录、容器、装载和网络名称空间。
sudo k0s stop
sudo k0s reset --data-dir=/data/k0s --kubelet-root-dir=/data/kubelet
# 查看 目录
ll /etc/systemd/system/ |grep k0s
ll /var/lib/k0s/
ll /var/lib/kubelet
ll /data/k0s/
ll /data/kubelet
# 删除 目录
sudo rm -rf /var/lib/kubelet
```

### 3. 所有节点必须重启（重要）

```properties
sudo reboot
```
### 4. 删除ETCD 所有数据（重要）
### 5. 强制删除pod

```properties
sudo k0s kubectl get node -owide
sudo k0s kubectl -n kube-system get pod -owide |grep 103
# 强制删除节点上的 Pod
sudo k0s kubectl drain app105 --ignore-daemonsets --delete-emptydir-data
# 强制删除 pod
sudo k0s kubectl -n kube-system delete pod <pod名> --force --grace-period=0
# 删除节点
sudo k0s kubectl delete node app105
```



# ■■■ 访问k8s

```properties
sudo cat /data/k0s/pki/admin.conf
mkdir -p /home/ekemp/.kube
sudo cp /data/k0s/pki/admin.conf /home/ekemp/.kube/config
# 查看集群状态
kubectl cluster-info
```

### 查看pod后台运行日志

```properties
k0s kubectl describe pod XXX -n kube-system
# 可以看到是镜像拉取失败的原因，所以需要开启魔法、开启代理
```

### ~~检查ETCD（外部跳过）~~

- SSL证书目录：/var/lib/k0s/pki

```properties
k0s etcd member-list
k0s kubectl get etcdmember
k0s kc get etcdmember
```
## 查看端口路由规则 ipvs / iptables 

- **注意netstat 查询不到端口：**netstat -tunlp |grep 30443

```properties
# ipvs 方式
sudo ipvsadm -Ln | grep 31180
# iptables 方式（跳过）
iptables -S -t nat|grep 30443
iptables -t nat -L KUBE-NODEPORTS -v --line-numbers
```

### 检查工作节点 calico网络

```properties
# 检查 type: calico-ipam
sudo cat /etc/cni/net.d/10-calico.conflist
sudo k0s kubectl -n kube-system get ds calico-node -oyaml
sudo k0s kubectl -n kube-system get po -owide | grep calico
sudo k0s kubectl -n kube-system exec calico-node-cbpfh -- calico-node -show-status
sudo k0s kubectl -n kube-system exec calico-node-cbpfh -- calico-node -status-reporter
# 获取配置文件环境变量
sudo k0s kubectl -n kube-system exec calico-node-cbpfh -- getconf -a
# 检查 birdcl
sudo k0s kubectl -n kube-system exec calico-node-cbpfh -- birdcl -v
# 查看 calico 的 bird 网络模式
route -n
# 输出不会出现 tunl0 或 vxlan.calico 设备
# ipvs 方式
sudo ipvsadm -Ln
```

### 正常 calico 状态

```properties
sudo k0s kubectl -n kube-system exec calico-node-cbpfh -- calico-node -show-status
#----------------------------------------------------------------
# bird v4 BGP peers   全局的（交换机）
+--------------+-----------+-------+------------+-------------+
| PEER ADDRESS | PEER TYPE | STATE |   SINCE    |  BGPSTATE   |
+--------------+-----------+-------+------------+-------------+
| 10.10.10.1   | Global    | up    | 2026-07-07 | Established |

# 修正后没有 tunl0 br-a86b7966e6a1
bird v4 routes
+-------------------+--------------+-----------+-------------------+---------+
|    DESTINATION    |   GATEWAY    |   IFACE   |    LEARNEDFROM    | PRIMARY |
+-------------------+--------------+-----------+-------------------+---------+
| 0.0.0.0/0         | 10.10.10.1   | bond0     | kernel1           | *       |
| 10.244.213.0/26   | 10.10.10.102 | bond0     | Mesh_10_10_10_102 | *       |
| 10.10.10.0/24     | N/A          | bond0     | direct1           | *       |
| 10.244.227.0/26   | 10.10.10.101 | bond0     | Mesh_10_10_10_101 | *       |
| 10.244.112.128/26 | 10.10.10.104 | bond0     | Mesh_10_10_10_104 | *       |
| 10.244.116.192/26 | 10.10.10.105 | bond0     | Mesh_10_10_10_105 | *       |
| 10.244.109.192/26 | N/A          | blackhole | static1           | *       |
+-------------------+--------------+-----------+-------------------+---------+
```

---

# ■■■ 修改k8s 配置文件地址

#### vim  /home/ekemp/.kube/config 

#### 地址修改成： 10.10.10.99:16443





---

# ■■■ 所有节点配置 containerd 国内源

**containerd不像docker，在/etc/docker/deamon.json文件配置一下insecure-registries就可以使用了，它的配置文件较复杂。**

- 官网CRI介绍：https://docs.k0sproject.io/v1.35.2+k0s.0/runtime/

- containerd配置：https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md

- containerd ctr 命令下载：https://github.com/containerd/containerd/releases

```properties
# 查看版本
sudo /data/k0s/bin/containerd -v
#--------------------------------------
containerd github.com/containerd/containerd 1.7.30 71c1c8666c6a999cc8c319160b6b2ea38c4a2c9e.m
# 下载的版本 多一个 v， hash 一致
containerd github.com/containerd/containerd v1.7.30 71c1c8666c6a999cc8c319160b6b2ea38c4a2c9e
```

#### 注意版本1.x配置只在工作节点有效

>
> registry.configs and registry.mirrors that were a part of containerd 1.4 are now DEPRECATED and will only be used if the config_path is not specified.
>

- **注意文件是否已存在，不然会覆盖掉原有的配置**

```properties
/data/k0s/bin/containerd config default > config-default.toml
```

- **注意默认文件地址不是  /etc/containerd/config.toml**

```properties
ps -ef |grep k0s
# ------原始路径-----------------------------------
/var/lib/k0s/bin/containerd --root=/var/lib/k0s/containerd --state=/run/k0s/containerd --address=/run/k0s/containerd.sock --log-level=info --config=/etc/k0s/containerd.toml
# ------自定义路径-----------------------------------
/data/k0s/bin/containerd --root=/data/k0s/containerd --state=/run/k0s/containerd --address=/run/k0s/containerd.sock --log-level=info --config=/etc/k0s/containerd.toml
```
### k0s 自动配置参数要关闭

- **重要参数：k0s_managed=false**
- k0s 允许动态配置 containerd CRI 运行时，其他配置可以放在/etc/k0s/containerd.d/ 目录下。详见https://docs.k0sproject.io/v1.35.2+k0s.0/runtime/#k0s-managed-dynamic-runtime-configuration

```properties
cat /etc/k0s/containerd.toml
cat /run/k0s/containerd-cri.toml
# 拷贝 编辑
sudo cp /run/k0s/containerd-cri.toml /etc/k0s/containerd.d/containerd-cri.toml
sudo vim /etc/k0s/containerd.toml
# ---输出------------------------------------------------------
k0s_managed=false
imports = ["/etc/k0s/containerd.d/containerd-cri.toml"]
version = 2
```
### 工作节点上修改containerd配置

```properties
cat /etc/k0s/containerd.d/containerd-cri.toml
sudo vim /etc/k0s/containerd.d/containerd-cri.toml
#---------最后几行修改 config_path 配置项--------------------------------------
...
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".registry]
    config_path = "/etc/containerd/certs.d"
```

### 配置公有仓库国内源

```properties
sudo mkdir -p /etc/containerd/certs.d/docker.io
# 创建hosts.toml
sudo tee /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.xuanyuan.me"
[host."https://docker.xuanyuan.me"]
  capabilities = ["pull", "resolve"]
EOF
# ----查看------------------
cat /etc/containerd/certs.d/docker.io/hosts.toml
```

### 配置私有仓库

- **一般不放开"push", 使用docker 专有socker的push ** 

```properties
sudo mkdir -p /etc/containerd/certs.d/10.10.10.102:5000
# 创建hosts.toml
sudo tee /etc/containerd/certs.d/10.10.10.102:5000/hosts.toml << EOF
server = "http://10.10.10.102:5000"
[host."http://10.10.10.102:5000"]
  capabilities = ["pull", "resolve"]
EOF
# ----查看------------------
cat /etc/containerd/certs.d/10.10.10.102:5000/hosts.toml
```

### 重启k0s服务生效

- k0s 绑定了特定版本的 containerd

```properties
# 控制节点
sudo systemctl restart k0scontroller
sudo systemctl status k0scontroller
# 工作节点
sudo systemctl restart k0sworker
sudo systemctl status k0sworker
```

---



#  控制3节点部署 nginx高可用

#### 下载最新版本  http://nginx.org/download

```properties
sudo mkdir -p /k0s/nginx/{conf,logs,sbin}
sudo cp nginx-prefix/sbin/nginx  /k0s/nginx/sbin/kube-nginx
sudo chmod a+x /k0s/nginx/sbin/kube-nginx
```
#### 配置文件
```properties
sudo tee /k0s/nginx/conf/kube-nginx.conf << EOF
    worker_processes auto;
    error_log /var/log/kube-nginx-error.log info;

    events {
        multi_accept on;
        use epoll;
        worker_connections  1024;
    }

    stream {
        upstream k8s_tcp {
            hash $remote_addr consistent;
            server 10.10.10.101:6443 max_fails=3 fail_timeout=30s;
            server 10.10.10.102:6443 max_fails=3 fail_timeout=30s;
            server 10.10.10.103:6443 max_fails=3 fail_timeout=30s;
        }

        server {
            listen 16443;
            proxy_connect_timeout 2s;
            proxy_timeout 120;
            proxy_pass k8s_tcp;
        }
    }
EOF
```
#### 配置 systemd unit 文件，启动服务

```properties
sudo tee /etc/systemd/system/kube-nginx.service << EOF
[Unit]
Description=k0s nginx proxy
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
ExecStartPre=/k0s/nginx/sbin/kube-nginx -c /k0s/nginx/conf/kube-nginx.conf -p /k0s/nginx -t
ExecStart=/k0s/nginx/sbin/kube-nginx -c /k0s/nginx/conf/kube-nginx.conf -p /k0s/nginx
ExecStop=/k0s/nginx/sbin/kube-nginx -c /k0s/nginx/conf/kube-nginx.conf -p /k0s/nginx -s quit
ExecReload=/k0s/nginx/sbin/kube-nginx -c /k0s/nginx/conf/kube-nginx.conf -p /k0s/nginx -s reload
PrivateTmp=true
Restart=always
RestartSec=5
StartLimitInterval=0
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
#### 启动 kube-nginx 服务：

```properties
sudo systemctl daemon-reload
sudo systemctl enable kube-nginx
sudo systemctl start kube-nginx
```

#### 检查 kube-nginx 服务运行状态

```properties
sudo systemctl status kube-nginx |grep 'Active:'
sudo netstat -anp |grep nginx | grep tcp
sudo systemctl is-enabled kube-nginx
```

确保状态为 `active (running)`，否则到 master 节点查看日志，排查原因：

```properties
journalctl -f -u kube-nginx
```





---
#  ■■■ 控制3节点部署 keepalived

### 下载安装 https://www.keepalived.org/download.html

```properties
sudo apt install -y gcc libssl-dev libpopt-dev
mkdir -p /k0s/keepalived && cd /k0s/keepalived
wget http://www.keepalived.org/software/keepalived-2.3.4.tar.gz
tar -zxvf keepalived-2.3.4.tar.gz && rm -f keepalived-2.3.4.tar.gz
cd keepalived-2.3.4 && mkdir keepalived-prefix
# --prefix：指定安装路径
./configure --prefix=$(pwd)/keepalived-prefix
# 编译安装
make && make install
# 复制环境变量配置
sudo cp keepalived/etc/sysconfig/keepalived /etc/default/
# 复制主配置文件
sudo mkdir /etc/keepalived
sudo cp etc/keepalived/keepalived.conf.sample /etc/keepalived/
# 复制可执行文件到 PATH
sudo cp bin/keepalived /usr/sbin/
# 修改 systemd 服务文件
sudo cp keepalived/keepalived.service ./
vim keepalived.service
# 复制 systemd 服务文件
sudo cp keepalived.service /usr/lib/systemd/system/
```
默认每隔3秒钟执行一次检测脚本，检查nginx服务是否启动，如果没启动就把nginx服务启动起来，如果启动不成功，就把keepalived服务down掉，让漂浮到备keepalived上

```properties
sudo tee /etc/keepalived/check_nginx.sh << EOF
#!/bin/bash
run=\$(ps -C kube-nginx --no-header | wc -l)
if [ \$run -eq 0 ]; then
  sudo systemctl stop kube-nginx
  sudo systemctl start kube-nginx
  sleep 3
  if [ \$(ps -C kube-nginx --no-header | wc -l) -eq 0 ]; then
    sudo systemctl stop keepalived
  fi
fi
EOF

# 修正执行权限
sudo chmod +x /etc/keepalived/check_nginx.sh
```

> 注意：检测脚本一定要写在vrrp_instance的前面也就是上面，而且花括号一定要有空格，追踪trace_script一定要在vip的后面，多少人栽在了这上面好多小时



### 【101】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID**：virtual_router_id 改成 99

```properties
sudo tee /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
   notification_email {
     java@ekemp.com.cn
   }
   notification_email_from 2355541806@qq.com
   smtp_server smtp.qq.com
   smtp_connect_timeout 30
   lvs_id K0S_API
}

vrrp_script chk_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state MASTER
    interface bond0
    virtual_router_id 99
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp2026
    }
    virtual_ipaddress {
        10.10.10.99/24
    }
    
    track_script {                      
        chk_nginx
    }
}
EOF
```

#### 【102】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID：virtual_router_id 改成 99**
- **修改priority：80**

```properties
sudo tee /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
   notification_email {
     java@ekemp.com.cn
   }
   notification_email_from 2355541806@qq.com
   smtp_server smtp.qq.com
   smtp_connect_timeout 30
   lvs_id K0S_API
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 99
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp2026
    }
    virtual_ipaddress {
        10.10.10.99/24
    }
    
    track_script {                      
        check_nginx
    }
}
EOF
```
#### 【103】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID：virtual_router_id 改成 99
- **修改priority：60**

```properties
sudo tee /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
   notification_email {
     java@ekemp.com.cn
   }
   notification_email_from 2355541806@qq.com
   smtp_server smtp.qq.com
   smtp_connect_timeout 30
   lvs_id K0S_API
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 99
    priority 60
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp2026
    }
    virtual_ipaddress {
        10.10.10.99/24
    }
    
    track_script {                      
        check_nginx
    }
}
EOF
```


#### 验证是否安装成功
```properties
sudo systemctl daemon-reload
sudo systemctl enable keepalived
sudo systemctl start keepalived

```

#### 查看日志文件
```properties
sudo systemctl status keepalived
sudo systemctl is-enabled keepalived 
tail -f -n 500 /var/log/messages
```





# 容器客户端工具比较

| 名称     | 容器管理 | pod支持 | 网络管理 | 存储卷管理 | 镜像构建 | 镜像管理 |
|---------|--------|--------|--------|---------|--------|--------|
| docker  | ✅     | ❌     | ✅     | ✅      | ✅     | ✅     |
| ctr     | ✅     | ❌     | ❌     | ❌      | ❌     | ✅     |
| crictl  | ✅     | ✅     | ❌     | ❌      | ❌     | ✅     |


| 操作           | docker                    | ctr                              | crictl                |
|--------------|---------------------------|----------------------------------|-----------------------|
| 显示正在运行的容器 | docker ps                 | ctr task ls / ctr containers ls  | crictl ps             |
| 启动容器        | docker run                | ctl run                          | crictl run            |
| 进入容器        | docker exec               | -                                | crictl exec           |
| 停止容器        | docker stop               | ctr pause                        | -    |
| 查看系统信息     | docker info               | -                                | crictl info           |
| 下载镜像        | docker pull               | ctr images pull                  | crictl pull           |
| 删除镜像        | docker rmi                | ctr image rm                     | crictl rmi            |
| 查看镜像列表     | docker images             | ctr image ls                     | crictl images  |
| 镜像导出        | docker save -o            | ctr image export                 | -                     |
| 镜像导入        | docker load -i            | ctr image import                 | -                     |



## ■■ 1. 安装ctr命令

- **containerd 原生内置调试工具，不支持 build**
- **有run 命令，也有 `container create` + `task start` 两步才等同于 run**
- **ctr客户端 主要区分了 3 个命名空间分别是k8s.io、moby和default**
- 【温馨提示】ctr images pull 拉取的镜像默认放在default，而 crictl pull 和 kubelet 默认拉取的镜像都在 k8s.io 命名空间下。所以通过ctr导入镜像的时候特别注意一点，最好指定命名空间。

```properties
cd /k0s/containerd/bin
chmod +x ctr
sudo cp ctr /usr/local/bin/
# 创建别名
vim ~/.bashrc
#----添加 ct ---------------------------
alias ct='sudo /usr/local/bin/ctr --address=/run/k0s/containerd.sock -n k8s.io'
#----生效--------------------------------
source ~/.profile

# 与 crictl image list 命令会显示相同的内容（只是格式不同）
ct image list
# 镜像导出
ct image export my-app.tar my-app:1.0.0
# 镜像导入
ct image import k8s-dashboard.tar
# 镜像tag
ct image tag 192.168.1.118:80/kubernetesui/dashboard:v2.7.0 10.10.10.102:80/kubernetesui/dashboard:v2.7.0
# http 镜像push --plain-http 或 --hosts-dir "/etc/containerd/certs.d"
ct image push --plain-http 10.10.10.102:80/kubernetesui/dashboard:v2.7.0
# 创建容器 
ct containers create nginx my-nginx
# 启动任务
ct tasks start my-nginx
# 查看任务
ct tasks ls
```



## ■■ 2.  安装 crictl 命令

- 下载crictl:  https://github.com/kubernetes-sigs/cri-tools/releases
- **crictl 调试k8s节点工具， 不支持 push build run** , 


```properties
cd /k0s/containerd/bin
chmod +x crictl
sudo cp crictl /usr/local/bin/crictl
```

### 创建配置文件

```properties
sudo ls -l /etc/crictl.yaml
# 创建环境变量文件
sudo tee /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/k0s/containerd.sock
image-endpoint: unix:///run/k0s/containerd.sock
timeout: 10
debug: false
EOF

# 创建别名
vim ~/.bashrc
#----添加 cc ---------------------------
alias cc='sudo crictl'
#----生效--------------------------------
source .profile
```

### 检查


```properties
cc config --list
cc pull 10.10.10.102:80/dev/ddap-mqtt-ws-service:202507292128
# 镜像相关操作：容器运行独立于镜像存储,与删除镜像无关
cc image list
cc rmi XXX
# 容器相关操作
cc ps -a
cc rm XXX
# Pod相关操作
cc pods list
cc rmp XXX
# 内存信息
cc stats
# 获取日志：
cc logs <container-id>
```



## ■■ 3.  安装 nerdctl 命令

- Rootless mode下载：https://github.com/containerd/nerdctl
- **用 containerd 子项目，兼容docker 可 build push run**
- **具备延迟拉取镜像（lazy-pulling）、镜像加密（imgcrypt）**
- **构建镜像需要安装 `buildctl` 并运行 `buildkitd`服务**：https://github.com/moby/buildkit
  - **服务端 buildkitd**：当前支持 runc 和 containerd 作为 worker，默认是 runc，k0s使用 containerd
  - **客户端 buildctl**：负责解析 Dockerfile，并向服务端 buildkitd 发出构建请求
  - buildkit 是典型的 C/S 架构，客户端和服务端是可以不在一台服务器上，而 nerdctl 在构建镜像的时候也作为 buildkitd 的客户端


```properties
cd /k0s/containerd/bin
chmod +x nerdctl
sudo cp nerdctl /usr/local/bin/nerdctl
```

### 创建配置文件

- 配置说明：https://github.com/containerd/nerdctl/blob/main/docs/config.md

```properties
sudo ls -l /etc/nerdctl
sudo mkdir -p /etc/nerdctl
# 创建 配置文件指向 k0s
sudo tee /etc/nerdctl/nerdctl.toml <<EOF
debug          = false
debug_full     = false
address        = "unix:///run/k0s/containerd.sock"
namespace      = "k8s.io"
EOF

# 创建别名
vim ~/.bashrc
#----添加 nd ---------------------------
alias nd='sudo nerdctl'
#----生效--------------------------------
source ~/.profile
```

### 命令与docker 相同


```properties
# 删除所有未使用的镜像
nd system prune -a
# 内存信息
nd stats
```

###  创建定时删除镜像垃圾文件

```properties
cat /k0s/containerd/clearImages.sh
# ----创建文件------------------------------------
sudo tee /k0s/containerd/clearImages.sh << 'EOF'
#!/bin/bash
# 获取当前年份和日期
YEAR=$(date +%Y)
DATE=$(date +%m-%d)

# 创建日志目录（年份/日期）
LOG_DIR="/data/logs/cron/clear-images/${YEAR}/${DATE}"
sudo mkdir -p "${LOG_DIR}"

# 执行清理命令，输出到日志文件
sudo nerdctl system prune -a -f 2>&1 | sudo tee -a "${LOG_DIR}/clear-images.log" > /dev/null
EOF
# ----测试----------------------------------------
/bin/bash /k0s/containerd/clearImages.sh
```

*   新建cron服务

```properties
# 编辑某个用户的cron服务
crontab -e
--------------------------------------------------------
# every Saturday 2:05 excute clear docker none image
5 2 * * 6 /bin/bash /k0s/containerd/clearImages.sh > /dev/null 2>&1
# 列出某个用户cron服务的详细内容 
crontab -l
# 设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数,
# 比如说root查看自己的cron设置: crontab -u root -l
crontab -u
```

---



## ■■ 4. docker CRI 适配

- **docker version: 29.3.0**
- **自带运行 /usr/bin/containerd 服务版本 v2.2.2，链接：/run/containerd/containerd.sock**
- **自带 /usr/bin/ctr 命令版本 v2.2.2，namespace = "moby"**

```properties
# 创建别名
vim ~/.bashrc
#----添加 ct ---------------------------
alias docker-ctr='sudo /usr/bin/ctr --address=/run/containerd/containerd.sock -n moby'
#----生效--------------------------------
source ~/.profile

# 查看镜像
docker-ctr images ls
# 查看容器
docker-ctr c ls
# 查看任务
docker-ctr tasks ls
#
docker-ctr prune
```

### 使用nerdctl

```properties
sudo ls -l /etc/nerdctl
sudo mkdir -p /etc/nerdctl
# 创建 配置文件指向 k0s
cat /etc/nerdctl/nerdctl.toml
sudo tee /etc/nerdctl/nerdctl.toml <<EOF
debug          = false
debug_full     = false
address = "unix:///run/containerd/containerd.sock"
namespace = "moby"
EOF
# 查看镜像
nd images
# 删除所有未使用的镜像
nd system prune -a
# docker 删除所有未使用的镜像
docker system prune -a
```

### 配置 daemon.json

> containerd(snapshotters) Docker Engine 29.0 及更高版本的默认设置。使用 containerd 快照器进行镜像存储，仅存一份数据（要不然要存两份）。支持多平台镜像和认证。启用 containerd 镜像存储后，overlay2 驱动程序中的现有镜像和容器仍保留在磁盘上，但会被隐藏。如果您切换回 overlay2，它们将重新出现。要将现有镜像与 containerd 镜像存储一起使用，请先将其推送到镜像仓库，或使用相关工具`docker save`导出它们。详见 https://docs.docker.com/engine/storage/containerd/

-  **"storage-driver": "overlayfs"**
- **新增 features**："containerd-snapshotter": true**

```properties
# 查看 存储驱动 overlay2 or containerd
docker info | grep "Storage Driver"
# 配置
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
      "https://docker.xuanyuan.me",
      "https://docker.tbedu.top"
   ],
  "insecure-registries": ["10.10.10.102:80"],
  "storage-driver": "overlayfs",
  "features": {
    "containerd-snapshotter": true
  },
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

### docker 29.0+ 使用 containerd 存储镜像

| 安装场景                  | 默认存储后端   | 存储位置                    | 行为                                                         |
| ------------------------- | -------------- | --------------------------- | ------------------------------------------------------------ |
| **全新安装 Docker 29.0+** | **containerd** | `/var/lib/containerd/`      | `docker images` 与 `ctr -n moby images ls` 看到的是**同一份数据**。 |
| **从旧版本升级**          | **overlay2**   | `/var/lib/docker/overlay2/` | 为保持兼容性，继续使用旧存储结构。可通过配置切换到 `containerd` 存储。原来**存两份数据** |








# ■■■ 安装k8s-dashboard版本2.7

### 下载安装

- 官网：https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

#### 在recommended.yaml里访问方式调整为nodeport

- 在Service 名为 kubernetes-dashboard
- 增加：type: NodePort
- 增加：nodePort: 30443
- 工作节点执行：iptables -S -t nat|grep 30443
- 访问工作节点：https://10.10.10.99:30443/

### 增加用户

#### k apply -f dashboard-adminuser.yaml 

```properties
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  annotations:
    kubernetes.io/service-account.name: admin-user
  name: admin-user
  namespace: kubernetes-dashboard
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURBRENDQWVpZ0F3SUJB---
  namespace: a3ViZXJuZXRlcy1kYXNoYm9hcmQ=
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNkluQlNXamMyUVhoRk5XZFRU---
```
### 创建token并绑定SA 和Secret

```properties
# 请求一个只绑定到SA，不绑定到 Secret 对象实例的令牌,365天过期
# kubectl create token admin-user --duration 8760h
# 请求一个绑定到 Secret 对象实例的令牌,365天过期
kubectl create token admin-user --bound-object-kind Secret --bound-object-name admin-user --duration 8760h -n kubernetes-dashboard
# -------测试----------------------------------------------
eyJhbGciOiJSUzI1NiIsImtpZCI6InZnZ3RHQUxMUnJkNFRUVXVEOVJCM1psOHh6NmktT1RCXzB3SHJWT2xHY2MifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIiwic3lzdGVtOmtvbm5lY3Rpdml0eS1zZXJ2ZXIiXSwiZXhwIjoxNzg0MzU3OTI4LCJpYXQiOjE3NTI4MjE5MjgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2YyIsImp0aSI6IjNkMjI3MzI2LTc4M2UtNDc4Yi1hZjlhLTc0YzRiYWJmNzM4OCIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoia3ViZXJuZXRlcy1kYXNoYm9hcmQiLCJzZWNyZXQiOnsibmFtZSI6ImFkbWluLXVzZXIiLCJ1aWQiOiJmMWQxNGZmYS1iYmI4LTRmNGItYjgwNC1mNWIyYmVhMWRiY2IifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImFkbWluLXVzZXIiLCJ1aWQiOiI2Mzc5ZGQ5OC03NjVlLTQ4MTgtOWUzYy02MjFjNzBiMGY3MzAifX0sIm5iZiI6MTc1MjgyMTkyOCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmFkbWluLXVzZXIifQ.Gp7A-c90eTHSfW2RWsw6oPbNn73sNw6ff2lxLdhi6PraPnVbo4NxelLYpzbPNr8PWKO3Ks_tQLOsLh3euSVi371JgMbsRPM-XvuYtfmK87B-O83_ioSTnlPkL6SbY06IAcNBRwFeD46s4pbM8L8QN5_e3EV-meTNhO1RSHMv_nCILlgLAFjc_ykUf3SMt1hu7y26918DnHqwqFClDP9E78ogm4jauxYBq0B90NmiYMTJSNM7WltQp8K9tGUrnJNvbjfHXwcu191yybKxcdiRKld-WcoG_NxJrvPp2CzqFzaZt9F53FVDGjx0ntSEcvgETbMTFPJeiGtXmUUpMJt-sg
# -------生产1----------------------------------------------
eyJhbGciOiJSUzI1NiIsImtpZCI6IkhTeXBoVWs0eGZ6eGZzdFpmS1dvWXdNT1JYVzNXLU5kTk9SOG5IcVpRSGsifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIiwic3lzdGVtOmtvbm5lY3Rpdml0eS1zZXJ2ZXIiXSwiZXhwIjoxODA2NDg1Mzk3LCJpYXQiOjE3NzQ5NDkzOTcsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2YyIsImp0aSI6ImUzZTQ3OTc1LWJhYTEtNGJjZi1iNDQ3LTBkMTlhOTY4MDg0YSIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoia3ViZXJuZXRlcy1kYXNoYm9hcmQiLCJzZWNyZXQiOnsibmFtZSI6ImFkbWluLXVzZXIiLCJ1aWQiOiIyZWI4NGIwZC00YmFjLTRlMWMtODdlYi0wZGViZjYwMTAzZjQifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImFkbWluLXVzZXIiLCJ1aWQiOiJhY2I5MzkxZC1mMmNhLTQ3NjgtYWJhZC02NzkwYzUzMjNjYzkifX0sIm5iZiI6MTc3NDk0OTM5Nywic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmFkbWluLXVzZXIifQ.LFcfVKGWz_hKlVI_M8EY1kJDLTsCYKvwGWgL_U7t3fi-xlgHhwZrd3HNG9M8sSdtlMEUjbb_Xg8YRmmpReIdAAdzph0t49LIa_g3ds5Z2Oh8v-aA-mcP3YDnjRKOzFeuWe2YyArdKOceQANbE_IiUXMDfd9ca2J1MR8qtXlZAsOVCn4j0WacE4JyAngqaPIDzAqM1I_iZSS3EC1HbY2qQ_m0wdcrVz7keXA1ijUeCtQpcQWQWdiiIF0y6feDVTAsob7wawyWHREtkKsErBYg6K4EaG7fcCLPjJj0czmXn48w01mzv658vpSN_ttKkvu89zVldNYNny0yd-Uu6mxIuA
```

### 此方法获取token可能无效

```properties
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

---

## 部署httpbin检查

```properties
curl http://httpbin:8000/get?show_env=true
curl http://10.10.10.128:8000/get?show_env=true
```

## 校验dns

```properties
kubectl -n httpbin run dns-tools --image=192.168.1.118:80/infoblox/dnstools --rm -it --restart=Never

nslookup api.develop.ekemp.com.cn
nslookup api-internal.develop.ekemp.com.cn
dig api.develop.ekemp.com.cn
dig api-internal.develop.ekemp.com.cn
```





# ■■■ 安装k8s-dashboard版本3.x（待定）







---


# ■■■ 安装helm（待定）

官网下载：https://github.com/helm/helm/releases

```properties
cd /k0s
# #在线安装
tar -zxvf  helm-v4.1.3-linux-amd64.tar.gz
cd linux-amd64
sudo cp helm /usr/bin/

# 查看helm版本
helm  version
# 查看帮助
helm  --help
```

### helm 常用命令

```properties
# 查看当前配置的仓库地址
$ helm repo list
# 删除默认仓库，默认在国外pull很慢
$ helm repo remove stable
# 添加几个常用的仓库,可自定义名字
helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo add kaiyuanshe http://mirror.kaiyuanshe.cn/kubernetes/charts
helm repo add azure http://mirror.azure.cn/kubernetes/charts
# 搜索chart
$ helm search repo apisix
# 拉取chart包到本地
$ helm pull bitnami/redis-cluster --version 8.1.2
# 安装redis-ha集群，取名redis-ha，需要指定持存储类
$ helm install redis-cluster bitnami/redis-cluster --set global.storageClass=nfs,global.redis.password=xiagao --version 8.1.2
# 卸载
$ helm uninstall redis-cluster
```





























