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
| **calico** | **v3.29.4-0** |      | **crictl** | **1.27.0** |
| **bird** | **v0.3.3+birdv1.6.8** |      | coredns | 1.2.6 |


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

### 所有节点下载安装k0s

```properties
cd /k0s/bin
# 下载k0s文件
wget https://k0sproject/k0s/releases/download/v1.35.2/k0s.0/k0s-v1.35.2+k0s.0-amd64
# 下载k0sctl文件
wget https:///k0sproject/k0sctl/releases/download/v0.28.0/k0sctl-linux-amd64
# 授执行权限
chmod +x k0s-v1.35.2+k0s.0-amd64
chmod +x k0sctl-linux-amd64
# 注意不能放在/usr/bin目录，不然后续会报错
sudo cp k0sctl-linux-amd64 /usr/local/bin/k0sctl
sudo cp k0s-v1.35.2+k0s.0-amd64 /usr/local/bin/k0s
# 查看版本
k0sctl version
k0s version
k0s sysinfo
```

### 查看集群需要的镜像（跳过）

```properties
k0s airgap list-images
# -----------------------------------------------------
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
cd /k0s && sudo k0sctl init > k0sctl.yaml
```
### vim /k0s/k0sctl.yaml

```properties
apiVersion: k0sctl.k0sproject.io/v1beta1
kind: Cluster
metadata:
  name: k0s-cluster
  user: admin
spec:
  hosts:
  - ssh:
      address: 10.10.10.101
      user: ekemp
      port: 22
      keyPath: /home/ekemp/.ssh/id_rsa
    role: controller+worker
    noTaints: true
    useExistingK0s: true
    dataDir: /data/k0s
    kubeletRootDir: /data/kubelet
    localhost:
      enabled: true
  - ssh:
      address: 10.10.10.102
      user: ekemp
      port: 22
      keyPath: /home/ekemp/.ssh/id_rsa
    role: controller+worker
    noTaints: true
    useExistingK0s: true
    dataDir: /data/k0s
    kubeletRootDir: /data/kubelet
  - ssh:
      address: 10.10.10.103
      user: ekemp
      port: 22
      keyPath: /home/ekemp/.ssh/id_rsa
    role: controller+worker
    noTaints: true
    useExistingK0s: true
    dataDir: /data/k0s
    kubeletRootDir: /data/kubelet
   - ssh:
      address: 10.10.10.104
      user: ekemp
      port: 22
      keyPath: /home/ekemp/.ssh/id_rsa
    role: worker
    useExistingK0s: true
    dataDir: /data/k0s
    kubeletRootDir: /data/kubelet
  - ssh:
      address: 10.10.10.105
      user: ekemp
      port: 22
      keyPath: /home/ekemp/.ssh/id_rsa
    role: worker
    useExistingK0s: true
    dataDir: /data/k0s
    kubeletRootDir: /data/kubelet
  k0s:
    version: v1.35.2+k0s.0
    config:
      apiVersion: k0s.k0sproject.io/v1beta1
      kind: ClusterConfig
      metadata:
        name: cx-k0s-cluster
      spec:
        images:
          repository: 10.10.10.102:80
          konnectivity:
            image: k0sproject/apiserver-network-proxy-agent
            version: v0.34.0-1       
          metricsserver:
            image: k0sproject/metrics-server
            version: v0.8.1-0
          coredns:
            image: k0sproject/coredns
            version: 1.14.2-0
          pause:
            image: k0sproject/pause
            version: 3.10.1
          kubeproxy:
            image: k0sproject/kube-proxy
            version: v1.35.2
          kuberouter:
            cni:
              image: k0sproject/kube-router
              version: v2.7.1-iptables1.8.11-0
            cniInstaller:
              image: k0sproject/cni-node
              version: 1.8.0-k0s.0                       
          calico:
            kubecontrollers:
              image: k0sproject/calico-kube-controllers
              version: v3.31.4-1
            cni:
              image: k0sproject/calico-cni
              version: v3.31.4-1
            node:
              image: k0sproject/calico-node
              version: v3.31.4-1
        telemetry:
          enabled: false
        api:
          k0sApiPort: 9443
          port: 6443
          sans:
            - 10.10.10.101
            - 10.10.10.102
            - 10.10.10.103
            - 10.10.10.99
          ca:
            expiresAfter: 876000h
            certificatesExpireAfter: 876000h
        network:
          provider: calico
          podCIDR: 10.244.0.0/16
          serviceCIDR: 10.96.0.0/12
          nodeLocalLoadBalancing:
            enabled: true
            type: EnvoyProxy
            envoyProxy:
              image:
                image: k0sproject/envoy-distroless
                version: v1.37.1
          calico:
            mode: bird
          kubeProxy:
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
  options:
    wait:
      enabled: true
    drain:
      enabled: true
      gracePeriod: 2m0s
      timeout: 5m0s
      force: true
      ignoreDaemonSets: true
      deleteEmptyDirData: true
      podSelector: ""
      skipWaitForDeleteTimeout: 0s
    concurrency:
      limit: 30
      workerDisruptionPercent: 10
      uploads: 5
    evictTaint:
      enabled: false
      taint: k0sctl.k0sproject.io/evict=true
      effect: NoExecute
      controllerWorkers: false
```

- **ETCD配置：**https://docs.k0sproject.io/v1.33.2+k0s.0/configuration/#specstorageetcdexternalcluster
- **calico配置：** https://docs.k0sproject.io/v1.33.2+k0s.0/configuration/#specnetworkcalico
- **calico官网配置：** https://docs.tigera.io/calico/latest/reference/configure-calico-node
- **calico mode = bird**，不是 **vxlan** 模式，也不是 **ipip** 模式（deprecated）
- **自动生成的SSL文件路径：/var/lib/k0s/pki**  或自定义的 /data/k0s/pki
- **SAN带IP列表：k0s-api.crt、server.crt、etcd/peer.crt（只带本机IP）**
- **spec.hosts[*].installFlags： k0s install 命令额外参数列表**



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



## ~~k0scontroller 启动参数(跳过)~~

```properties
vim /etc/systemd/system/k0scontroller.service
#--------------------------------------------------
ExecStart=/usr/local/bin/k0s controller --config=/etc/k0s/k0s.yaml --data-dir=/var/lib/k0s --enable-worker=true
```

### k0s controller命令参数详解

- 别名：k0s install controller 或 k0s server，查看参数：k0s controller --help

- 默认配置文件：--config   /etc/k0s/k0s.yaml

- 默认数据目录：--data-dir  /var/lib/k0s

- **在 hosts[*].installFlags中 打开动态配置参数：--enable-dynamic-config**

- ##### `spec.k0s.dynamicConfig=true` 启用 k0s 动态配置。如果满足以下条件，该设置将自动设置为 true：

  - 任何控制节点 installFlags 都有 `--enable-dynamic-config`
  - 任何现有的控制器节点都有`--enable-dynamic-config`运行参数（可通过`k0s status -o yaml`查看）

- kube-controller-manager-额外参数：--kube-controller-manager-extra-args

### 可禁用的组件： --disable-components

- applier-manager   autopilot   control-api     **coredns**     csr-approver   endpoint-reconciler  helm 
- **konnectivity-server**   **kube-controller-manager**   **kube-proxy**   **kube-scheduler**   **metrics-server**
- network-provider   node-role   system-rbac   windows-node    worker-config



---

# ■■■ 部署安装


```properties
sudo cp /k0s/bin/k0sctl-linux-amd64 /usr/local/bin/k0sctl
sudo cp /k0s/bin/k0s-v1.35.2+k0s.0-amd64 /usr/local/bin/k0s
# 部署命令
cd /k0s && k0sctl apply --config k0sctl.yaml
#--------Finished-----------------------------
INFO ==> Finished in 1m12s               
INFO k0s cluster version v1.33.2+k0s.0 is now installed 
INFO Tip: To access the cluster you can now fetch the admin kubeconfig using: 
INFO      k0sctl kubeconfig  
#--------Failed log-------------------
ls -l /home/ekemp/.cache/k0sctl/
tail -f -n 500 /home/ekemp/.cache/k0sctl/k0sctl.log

# 每个节点配置查看
k0s status -o yaml
```

### 检查

```properties
sudo k0s kubectl cluster-info
sudo k0s kubectl get node -owide --show-labels
sudo k0s kubectl get pods -n kube-system
# controller 节点
journalctl -u k0scontroller -n 1000
# worker 节点
journalctl -u k0sworker -f 

sudo k0s kubectl -n kube-system get po
# 检查 Kubernetes Service
sudo k0s kubectl -n default get svc kubernetes
# 检查 Endpoints
sudo k0s kubectl -n default get endpoints kubernetes -n default
# 查看详细信息
sudo k0s kubectl -n default describe svc kubernetes -n default

# 各节点检查 k0s 配置
sudo vim /etc/k0s/k0s.yaml
```





---

# ■■■ 停止删除服务

### 1. 先检查主节点（同上）

### 2. 删除所有节点

```properties
# 清除已安装的系统服务、数据目录、容器、装载和网络名称空间。
cd /k0s && k0sctl reset --config k0sctl.yaml
# 删除旧配置节点
k0sctl reset --config k0sctl-old.yaml
# 清理 containerd 中的 k8s 数据
sudo ctr --address=/run/k0s/containerd.sock -n k8s.io container delete
sudo ctr --address=/run/k0s/containerd.sock -n k8s.io image ls
sudo ctr --address=/run/k0s/containerd.sock -n k8s.io image delete --all
```

### 2. （重要）所有节点停止受影响的服务
```properties
systemctl stop keepalived
```
### 3. 失败节点删除（跳过，若前一步有问题时使用）
```properties
k0s stop
ps -ef |grep kube
ps -ef |grep k0s
k0s reset
# 查看
ll /etc/systemd/system/ |grep k0s
ll /var/lib/k0s/
ll /var/lib/kubelet
ll /data/k0s/
ll /data/kubelet
# 删除
sudo rm -rf /var/lib/k0s
sudo rm -rf /var/lib/kubelet
sudo rm -rf /data/k0s
sudo rm -rf /data/kubelet
```
### 4. 删除主节点k0scontroller

```properties
systemctl daemon-reload && systemctl stop k0scontroller
systemctl disable k0scontroller
systemctl status k0scontroller
rm -f /etc/systemd/system/k0scontroller.service
```


### 6. 所有节点必须重启（重要）

```properties
reboot
```





---

# ■■■ 访问k8s

```properties
ll /data/k0s/pki/
cd /k0s && k0sctl kubeconfig > /k0s/admin.conf
# 此配置文件即 kubectl 配置文件
mkdir -p /home/ekemp/.kube
cp /k0s/admin.conf /home/ekemp/.kube/config
# 查看集群状态
sudo k0s kubectl cluster-info
```

### 查看报错日志
```properties
tail -f -n 500 /home/ekemp/.cache/k0sctl/k0sctl.log
```


### 查看集群node/pod状态

```properties
sudo k0s kubectl get node -o wide
sudo k0s kubectl get pod -A -o wide
```

### 查看pod后台运行日志

```properties
k0s kubectl describe pod XXX -n kube-system
# 可以看到是镜像拉取失败的原因，所以需要开启魔法、开启代理
```

### 查看控制节点k0s进程

- k0s controller 是父进程，其它8个都是子进程
- worker节点多了以容器运行的 kube-proxy 子进程

```properties
ps -ef |grep k0s
# --------------------------------------------
root       1251      1  /usr/local/bin/k0s controller --config=/etc/k0s/k0s.yaml
etcd       1512   1251  /var/lib/k0s/bin/etcd 
kube-ap+   1681   1251  /var/lib/k0s/bin/kube-apiserver 
root       1720   1251  /usr/local/bin/k0s api
root       1749   1251  /var/lib/k0s/bin/containerd --root=/var/lib/k0s/containerd
kube-sc+   1750   1251  /var/lib/k0s/bin/kube-scheduler --bind-address=127.0.0.1 
kube-ap+   1751   1251  /var/lib/k0s/bin/kube-controller-manager 
root       1783   1251  /var/lib/k0s/bin/kubelet 
konnect+   1795   1251  /var/lib/k0s/bin/konnectivity-server --mode=grpc 
# worker节点多了以容器运行的kube-proxy进程
root      68976  68820  /usr/local/bin/kube-proxy
```

### 查看工作节点k0s进程

- k0s worker -h
- k0s是父进程，containerd 和 kubelet 两个是子进程
- 多了以容器运行的 kube-proxy 子进程

```properties
root  27892      1  /usr/local/bin/k0s worker --data-dir=/var/lib/k0s worker --token
root  27937  27892  /var/lib/k0s/bin/containerd --root=/var/lib/k0s/containerd
root  27951  27892  /var/lib/k0s/bin/kubelet --cert-dir=/var/lib/k0s/kubelet/pki 
root  58023  57956  /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf 
```

### ~~检查ETCD（外部跳过）~~

- SSL证书目录：/var/lib/k0s/pki

```properties
k0s etcd member-list
k0s kubectl get etcdmember
k0s kc get etcdmember
```
### 检查工作节点 calico网络

```properties
# 检查 type: calico-ipam
sudo cat /etc/cni/net.d/10-calico.conflist
sudo k0s kubectl -n kube-system get ds calico-node -oyaml
sudo k0s kubectl -n kube-system get po -owide | grep calico
sudo k0s kubectl -n kube-system exec calico-node-bl7sb -- calico-node -show-status
sudo k0s kubectl -n kube-system exec calico-node-bl7sb -- calico-node -status-reporter
# 获取配置文件环境变量
sudo k0s kubectl -n kube-system exec calico-node-bl7sb -- getconf -a

# 检查 birdcl
sudo k0s kubectl -n kube-system exec calico-node-bl7sb -- birdcl -v

# 查看 calico 的 bird 网络模式
route -n
# 输出不会出现 tunl0 或 vxlan.calico 设备
```

###  修改 calico mode=bird（跳过）

```properties
sudo k0s kubectl get ippools
# 修改 Calico 配置ipipMode: Never,vxlanMode: Never
sudo k0s kubectl edit ippool
# 更改配置重启 calico-node
sudo k0s kubectl -n kube-system rollout restart ds calico-node
```



```properties
sudo k0s kubectl -n kube-system exec calico-node-bl7sb -- calico-node -show-status
#----------------------------------------------------------------
# 修正后没有 tunl0 br-a86b7966e6a1
+-------------------+--------------+-----------------+-------------------+---------+
|    DESTINATION    |   GATEWAY    |      IFACE      |    LEARNEDFROM    | PRIMARY |
+-------------------+--------------+-----------------+-------------------+---------+
| 0.0.0.0/0         | 10.10.10.1   | bond0           | kernel1           | *       |
| 10.244.213.0/26   | 10.10.10.102 | bond0           | Mesh_10_10_10_102 | *       |
| 10.10.10.0/24     | N/A          | bond0           | direct1           | *       |
| 10.244.227.0/26   | 10.10.10.101 | bond0           | Mesh_10_10_10_101 | *       |
| 10.244.112.128/26 | 10.10.10.104 | bond0           | Mesh_10_10_10_104 | *       |
| 10.244.116.192/26 | 10.10.10.105 | bond0           | Mesh_10_10_10_105 | *       |
| 10.244.109.193/32 | N/A          | cali2df39e47992 | kernel1           | *       |
| 10.244.109.192/26 | N/A          | blackhole       | static1           | *       |
+-------------------+--------------+-----------------+-------------------+---------+
```





---

#  入口3节点部署 nginx高可用

#### 源码编译安装nginx
```properties
mkdir -p /k8s/nginx/{conf,logs,sbin}
cp /k8s/nginx/nginx-1.29.0/nginx-prefix/sbin/nginx  /k8s/nginx/sbin/kube-nginx
chmod a+x /k8s/nginx/sbin/*
```
#### 配置文件
```properties
cat << EOF | tee /k8s/nginx/conf/kube-nginx.conf
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
            server 172.17.0.20:6443 max_fails=3 fail_timeout=30s;
            server 172.17.0.21:6443 max_fails=3 fail_timeout=30s;
            server 172.17.0.22:6443 max_fails=3 fail_timeout=30s;
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
cat << EOF | tee /etc/systemd/system/kube-nginx.service
[Unit]
Description=k8s nginx proxy
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
ExecStartPre=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx -t
ExecStart=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx
ExecStop=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx -s quit
ExecReload=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx -s reload
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
systemctl daemon-reload && systemctl enable kube-nginx && systemctl start kube-nginx
```

#### 检查 kube-nginx 服务运行状态

```properties
systemctl status kube-nginx |grep 'Active:'
netstat -anp |grep nginx
```

确保状态为 `active (running)`，否则到 master 节点查看日志，排查原因：

```properties
journalctl -f -u kube-nginx
```





---
#  ■■■ 入口3节点部署 keepalived

### 入口3节点安装

```properties
yum install -y gcc openssl-devel popt-devel
mkdir -p /k0s/keepalived && cd /k0s/keepalived
# 查看最新版本号：https://www.keepalived.org/download.html
wget http://www.keepalived.org/software/keepalived-2.3.4.tar.gz
tar -zxvf keepalived-2.3.4.tar.gz && rm -f keepalived-2.3.4.tar.gz
cd keepalived-2.3.4 && mkdir keepalived-prefix
# --prefix：指定安装路径
./configure --prefix=$(pwd)/keepalived-prefix
# 安装
make && make install
#复制文件
cp keepalived/etc/init.d/keepalived /etc/init.d
cp keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cd keepalived-prefix/ && mkdir /etc/keepalived
cp etc/keepalived/keepalived.conf.sample /etc/keepalived/
```
默认每隔3秒钟执行一次检测脚本，检查nginx服务是否启动，如果没启动就把nginx服务启动起来，如果启动不成功，就把keepalived服务down掉，让漂浮到备keepalived上

```properties
cat << EOF | tee /etc/keepalived/check_nginx.sh
#!/bin/bash
run=\$(ps -C kube-nginx --no-header | wc -l)
if [ \$run -eq 0 ]; then
  systemctl stop kube-nginx
  systemctl start kube-nginx
  sleep 3
  if [ \$(ps -C kube-nginx --no-header | wc -l) ]; then
    killall keepalived
  fi
fi
EOF
```

> 注意：检测脚本一定要写在vrrp_instance的前面也就是上面，而且花括号一定要有空格，追踪trace_script一定要在vip的后面，多少人栽在了这上面好多小时



### 【21】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID：virtual_router_id 改成 88**

```properties
cat << EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     pantianjun@changxing28.com
   }
   notification_email_from keepalived@changxing28.com
   smtp_server 172.17.0.110
   smtp_connect_timeout 30
   router_id emqx@172.17.0.30
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 88
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.88/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```

#### 【22】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID：virtual_router_id 改成 88**
- **修改priority：80**

```properties
cat << EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     pantianjun@changxing28.com
   }
   notification_email_from keepalived@changxing28.com
   smtp_server 172.17.0.110
   smtp_connect_timeout 30
   router_id emqx@172.17.0.32
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    virtual_router_id 88
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.88/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```
#### 【23】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID：virtual_router_id 改成 88**
- **修改priority：60**

```properties
cat << EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     pantianjun@changxing28.com
   }
   notification_email_from keepalived@changxing28.com
   smtp_server 172.17.0.110
   smtp_connect_timeout 30
   router_id emqx@172.17.0.32
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    virtual_router_id 88
    priority 60
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.88/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```


#### 验证是否安装成功
```properties
systemctl daemon-reload && systemctl enable keepalived && systemctl start keepalived
systemctl is-enabled keepalived 
```

#### 查看日志文件
```properties
systemctl status keepalived
tail -f -n 500 /var/log/messages
```





---

# ■■■ 修改k8s 配置文件地址

#### vim  /home/ekemp/.kube/config 

#### 地址修改成： 10.10.10.99:16443





---

# ■■■ 所有节点配置 containerd 国内源

**containerd不像docker，在/etc/docker/deamon.json文件配置一下insecure-registries就可以使用了，它的配置文件较复杂。**

https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md

下载containerd ctr 命令：https://github.com/containerd/containerd/releases

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
...
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

```properties
sudo mkdir -p /etc/containerd/certs.d/10.10.10.102:80
# 创建hosts.toml
sudo tee /etc/containerd/certs.d/10.10.10.102:80/hosts.toml << EOF
server = "http://10.10.10.102:80"
[host."http://10.10.10.102:80"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF
# ----查看------------------
cat /etc/containerd/certs.d/10.10.10.102:80/hosts.toml
```

### 查看最新配置

```properties
cd /k0s/containerd
sudo /data/k0s/bin/containerd config dump > combined.toml
sudo /data/k0s/bin/containerd config default > default.toml
diff default.toml combined.toml
```

### 安装ctr命令

- `ctr`：containerd 原生调试工具，直接与 containerd 交互

```properties
cd /k0s/containerd/bin
chmod +x ctr
sudo cp ctr /usr/local/bin/
# 与 crictl image list 命令会显示相同的内容（只是格式不同）
sudo ctr --address=/run/k0s/containerd.sock -n k8s.io i list
```

### 安装 crictl 命令

- `crictl`：Kubernetes CRI（容器运行时接口）工具，专为 Kubernetes 设计
- 

下载crictl:  https://github.com/kubernetes-sigs/cri-tools/releases

```properties
cd /k0s/containerd/bin
chmod +x crictl
sudo cp crictl /usr/local/bin/crictl
```

### 检查


```properties
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock config --list
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock pull 10.10.10.102:80/dev/ddap-mqtt-ws-service:202507292128
# 镜像相关操作
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock image list
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock rmi ~
# 容器相关操作
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock ps -a
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock rm ~
# Pod相关操作
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock pods list
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock rmp ~
# 内存信息
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock stats
# 获取日志：
sudo crictl --runtime-endpoint=unix:///run/k0s/containerd.sock logs <container-id>
```

### ~~重启containerd服务（跳过）~~

```properties
systemctl restart containerd
systemctl status containerd
journalctl -f -u containerd
ps -ef |grep containerd
/var/lib/k0s/bin/containerd config dump |grep certs
```

## 重启k0s服务

```properties
systemctl restart k0sworker
systemctl status k0sworker
```

## 查看端口路由规则 ipvs / iptables 

- **注意netstat 查询不到端口：**netstat -tunlp |grep 30443

```properties
# ipvs 方式
ipvsadm -Ln | grep 31180
# iptables 方式（跳过）
iptables -S -t nat|grep 30443
iptables -t nat -L KUBE-NODEPORTS -v --line-numbers
```





---

# ■■■ 安装k8s-dashboard版本2.7

### 下载安装

- 官网：https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

#### 在recommended.yaml里访问方式调整为nodeport

- 在Service 名为 kubernetes-dashboard
- 增加：type: NodePort
- 增加：nodePort: 30443
- 工作节点执行：iptables -S -t nat|grep 30443
- 访问工作节点：https://172.17.0.22:30443/

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
# -----------------------------------------------------
eyJhbGciOiJSUzI1NiIsImtpZCI6InZnZ3RHQUxMUnJkNFRUVXVEOVJCM1psOHh6NmktT1RCXzB3SHJWT2xHY2MifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjIiwic3lzdGVtOmtvbm5lY3Rpdml0eS1zZXJ2ZXIiXSwiZXhwIjoxNzg0MzU3OTI4LCJpYXQiOjE3NTI4MjE5MjgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2YyIsImp0aSI6IjNkMjI3MzI2LTc4M2UtNDc4Yi1hZjlhLTc0YzRiYWJmNzM4OCIsImt1YmVybmV0ZXMuaW8iOnsibmFtZXNwYWNlIjoia3ViZXJuZXRlcy1kYXNoYm9hcmQiLCJzZWNyZXQiOnsibmFtZSI6ImFkbWluLXVzZXIiLCJ1aWQiOiJmMWQxNGZmYS1iYmI4LTRmNGItYjgwNC1mNWIyYmVhMWRiY2IifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImFkbWluLXVzZXIiLCJ1aWQiOiI2Mzc5ZGQ5OC03NjVlLTQ4MTgtOWUzYy02MjFjNzBiMGY3MzAifX0sIm5iZiI6MTc1MjgyMTkyOCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmFkbWluLXVzZXIifQ.Gp7A-c90eTHSfW2RWsw6oPbNn73sNw6ff2lxLdhi6PraPnVbo4NxelLYpzbPNr8PWKO3Ks_tQLOsLh3euSVi371JgMbsRPM-XvuYtfmK87B-O83_ioSTnlPkL6SbY06IAcNBRwFeD46s4pbM8L8QN5_e3EV-meTNhO1RSHMv_nCILlgLAFjc_ykUf3SMt1hu7y26918DnHqwqFClDP9E78ogm4jauxYBq0B90NmiYMTJSNM7WltQp8K9tGUrnJNvbjfHXwcu191yybKxcdiRKld-WcoG_NxJrvPp2CzqFzaZt9F53FVDGjx0ntSEcvgETbMTFPJeiGtXmUUpMJt-sg
```

### 此方法获取token可能无效

```properties
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

---

# ■■■ 安装k8s-dashboard版本3.x（待定）







---


# ■■■ 安装helm（待定）

官网下载：https://github.com/helm/helm/releases

```properties
cd /k0s
# #在线安装
tar -zxvf  helm-v3.18.4-linux-amd64.tar.gz
cd linux-amd64
mv helm /usr/bin/

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





























