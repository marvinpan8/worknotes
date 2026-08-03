# k0s 集群部署-CentOS-7.9

官方文档地址  https://docs.k0sproject.io/v1.33.2+k0s.0/configuration/
系统部署要求  https://docs.k0sproject.io/v1.33.2+k0s.0/system-requirements/

## 版本说明

| 组件       | 版本号           |      | 组件           | 版本号       |
| ---------- | ---------------- | ---- | -------------- | ------------ |
| **centos** | **7.9.2009**     |      | dashboard      | 1.10.1       |
| **k0s** | **v1.33.2+k0s.0** |      | metrics-server | 0.3.1        |
| **k8s**     | **v1.33.2+k0s** |      | **containerd** | **1.6.33** |
| **etcd**   | **3.5.21**       |      | **ctr** | **1.6.33** |
| **calico** | **v3.29.4-0** |      | **crictl** | **1.27.0** |
| **bird** | **v0.3.3+birdv1.6.8** |      | coredns | 1.2.6 |


## 主机名

设置永久主机名称，然后重新登录:

```properties
# 将 m7-autocv-gpu01 替换为当前主机名
hostnamectl set-hostname k0s20
cat /etc/hostname
```

*   设置的主机名保存在 `/etc/hostname` 文件中；

- vim /etc/hosts

```properties
172.17.0.20 k0s20
172.17.0.21 k0s21
172.17.0.22 k0s22
172.17.0.23 k0s23
```

### 所有节点下载安装k0s

```properties
# 下载k0s文件
wget https://k0sproject/k0s/releases/download/v1.33.2/k0s.0/k0s-v1.33.2+k0s.0-amd64
# 下载k0sctl文件
wget https:///k0sproject/k0sctl/releases/download/v0.25.1/k0sctl-linux-amd64
# 授执行权限
chmod +x k0s-v1.33.2+k0s.0-amd64
chmod +x k0sctl-linux-amd64
# 注意不能放在/usr/bin目录，不然后续会报错
mv k0sctl-linux-amd64 /usr/local/bin/k0sctl
mv k0s-v1.33.2+k0s.0-amd64 /usr/local/bin/k0s

k0sctl version
k0s version
```

### 查看集群需要的镜像（跳过）

```properties
k0s airgap list-images
# -----------------------------------------------------
quay.io/k0sproject/calico-cni:v3.29.4-0
quay.io/k0sproject/calico-kube-controllers:v3.29.4-0
quay.io/k0sproject/calico-node:v3.29.4-0
quay.io/k0sproject/coredns:1.12.2
quay.io/k0sproject/apiserver-network-proxy-agent:v0.32.0
quay.io/k0sproject/kube-proxy:v1.33.2
quay.io/k0sproject/kube-router:v2.5.0-iptables1.8.11-0
quay.io/k0sproject/cni-node:1.7.1-k0s.0
quay.io/k0sproject/metrics-server:v0.7.2-0
quay.io/k0sproject/pause:3.10.1
```





---

# 【40节点】生成配置文件

```properties
cd /k0s && k0sctl init > k0sctl.yaml
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
      address: 172.17.0.20
      user: root
      port: 22
      keyPath: ~/.ssh/id_rsa
    role: controller+worker
  - ssh:
      address: 172.17.0.21
      user: root
      port: 22
      keyPath: ~/.ssh/id_rsa
    role: controller+worker
  - ssh:
      address: 172.17.0.22
      user: root
      port: 22
      keyPath: ~/.ssh/id_rsa
    role: controller+worker
  - ssh:
      address: 172.17.0.23
      user: root
      port: 22
      keyPath: ~/.ssh/id_rsa
    role: worker
  k0s:
    config:
      apiVersion: k0s.k0sproject.io/v1beta1
      kind: ClusterConfig
      metadata:
        name: cx-k0s-cluster
      spec:
        telemetry:
          enabled: false
        api:
          k0sApiPort: 9443
          port: 6443
          sans:
          - 172.17.0.20
          - 172.17.0.21
          - 172.17.0.22
          - 172.17.0.88
          - 172.17.0.252
          ca:
            expiresAfter: 87600h
            certificatesExpireAfter: 87600h
        network:
          nodeLocalLoadBalancing:
            enabled: true
            type: EnvoyProxy
          provider: calico
          calico:
            mode: bird
          podCIDR: 10.244.0.0/16
          serviceCIDR: 10.96.0.0/12
          clusterDomain: cluster.local
          kubeProxy:
            mode: ipvs
        storage:
          etcd:
            ca:
              expiresAfter: 87600h
              certificatesExpireAfter: 87600h
          type: etcd
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
- **SSL文件路径：/var/lib/k0s/pki**
- **SAN带IP列表：k0s-api.crt、server.crt、etcd/peer.crt（只带本机IP）**
- **spec.hosts[*].installFlags： k0s install 命令额外参数列表**



### 【40节点】配置ssh免密登录其他节点

```properties
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa
sshpass -p 'Abcd.1234' ssh-copy-id -o StrictHostKeyChecking=no root@172.17.0.20
sshpass -p 'Abcd.1234' ssh-copy-id -o StrictHostKeyChecking=no root@172.17.0.21
sshpass -p 'Abcd.1234' ssh-copy-id -o StrictHostKeyChecking=no root@172.17.0.22
sshpass -p 'Abcd.1234' ssh-copy-id -o StrictHostKeyChecking=no root@172.17.0.23
cat ~/.ssh/known_hosts
```



## k0scontroller 系统启动参数

```properties
vim /etc/systemd/system/k0scontroller.service
#--------------------------------------------------
ExecStart=/usr/local/bin/k0s controller --config=/etc/k0s/k0s.yaml --data-dir=/var/lib/k0s --enable-worker=true
```

## k0s controller命令参数详解

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
cd /k0s && k0sctl apply --config k0sctl.yaml
--------Finished-------------------
INFO ==> Finished in 1m12s               
INFO k0s cluster version v1.33.2+k0s.0 is now installed 
INFO Tip: To access the cluster you can now fetch the admin kubeconfig using: 
INFO      k0sctl kubeconfig  
#--------failed-------------------
tail -f -n 500 /root/.cache/k0sctl/k0sctl.log

# 每个节点配置查看
k0s status -o yaml
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
```

### 2. 所有节点停止受影响的服务（重要）
```properties
systemctl stop keepalived
```
### 3. 失败节点删除（跳过，若前一步有问题时使用）
```properties
k0s stop
ps -ef |grep kube
ps -ef |grep k0s
k0s reset
ll /etc/systemd/system/ |grep k0s
ll /var/lib/k0s/pki
rm -rf /var/lib/k0s/pki/*
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
ll /var/lib/k0s/pki/
cd /k0s && k0sctl kubeconfig > /var/lib/k0s/pki/admin.conf
# 此配置文件即 kubectl 配置文件
cp /var/lib/k0s/pki/admin.conf /root/.kube/config
```

### 查看报错日志
```properties
tail -f -n 500 /root/.cache/k0sctl/k0sctl.log
```


### 查看集群node/pod状态

```properties
# 注意不显示主节点
k0s kubectl get node -o wide
k0s kubectl get pod -A -o wide
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

### 检查ETCD

- SSL证书目录：/var/lib/k0s/pki

```properties
k0s etcd member-list
k0s kubectl get etcdmember
k0s kc get etcdmember
```
### 检查工作节点 calico网络

```properties
# 检查 type: calico-ipam
cat /etc/cni/net.d/10-calico.conflist
kn kube-system
k get ds calico-node -oyaml
k get po -owide
k exec calico-node-gfwpm -- calico-node -show-status
k exec calico-node-gfwpm -- calico-node -status-reporter
# 获取配置文件环境变量
k exec calico-node-gfwpm -- getconf -a

# 检查 birdcl
k exec calico-node-gfwpm -- birdcl -v

# 查看 calico 的 bird 网络模式
route -n
# 输出不会出现 tunl0 或 vxlan.calico 设备
```

###  修改 calico mode=bird（跳过）

```properties
kubectl get ippools
# 修改 Calico 配置ipipMode: Never,vxlanMode: Never
kubectl edit ippool
# 更改配置重启 calico-node
kubectl rollout restart ds calico-node
```





---

#  入口3节点部署 nginx高可用【21/22/23】

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
#  ■■■ 入口3节点部署 keepalived【21/22/23】

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

#### vim  /root/.kube/config 

#### 地址修改成： 172.17.0.88:16443





---

# ■■■ 所有节点配置 containerd 国内源

**containerd不像docker，在/etc/docker/deamon.json文件配置一下insecure-registries就可以使用了，它的配置文件较复杂。**

```properties
# 查看版本
containerd -v
#--------------------------------------
containerd containerd.io 1.6.33 d2d58213f83a351ca8f528a95fbd145f5654e957
```

#### 注意版本1.x配置只在工作节点有效

>
> registry.configs and registry.mirrors that were a part of containerd 1.4 are now DEPRECATED and will only be used if the config_path is not specified.
>

- **注意文件是否已存在，不然会覆盖掉原有的配置**

```properties
containerd config default > config-default.toml
```

- **注意默认文件地址不是  /etc/containerd/config.toml**

```properties
ps -ef |grep k0s
# -----------------------------------------
/var/lib/k0s/bin/containerd --root=/var/lib/k0s/containerd --state=/run/k0s/containerd --address=/run/k0s/containerd.sock --log-level=info --config=/etc/k0s/containerd.toml
```
### k0s 自动配置参数要关闭

- **重要参数：k0s_managed=false**

```properties
cat /etc/k0s/containerd.toml
cat /run/k0s/containerd-cri.toml
cp  /run/k0s/containerd-cri.toml  /etc/k0s/containerd.d/containerd-cri.toml
vim /etc/k0s/containerd.toml
# ---输出------------------------------------------------------
k0s_managed=false
imports = ["/etc/k0s/containerd.d/containerd-cri.toml"]
version = 2
```
### 工作节点上修改containerd配置

```properties
cat /etc/k0s/containerd.d/containerd-cri.toml
vim /etc/k0s/containerd.d/containerd-cri.toml
#---------修改 config_path 配置项--------------------------------------
...
[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".registry]
    config_path = "/etc/containerd/certs.d"
...
```

### 配置公有仓库国内源

```properties
mkdir -p /etc/containerd/certs.d/docker.io
# 创建hosts.toml
cat << EOF | tee /etc/containerd/certs.d/docker.io/hosts.toml
server = "https://docker.xuanyuan.me"
[host."https://docker.xuanyuan.me"]
  capabilities = ["pull", "resolve"]
EOF
# ----查看------------------
cat /etc/containerd/certs.d/docker.io/hosts.toml
```

### 配置私有仓库

```properties
mkdir -p /etc/containerd/certs.d/172.17.0.15:85
# 创建hosts.toml
cat << EOF | tee /etc/containerd/certs.d/172.17.0.15:85/hosts.toml
server = "http://172.17.0.15:85"
[host."http://172.17.0.15:85"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF
# ----查看------------------
cat /etc/containerd/certs.d/172.17.0.15:85/hosts.toml
```

### 查看最新配置

```properties
containerd config dump > combined.toml
containerd config default > default.toml
diff default.toml combined.toml
```

### 安装ctr命令
```properties
ctr --address=/run/k0s/containerd.sock -n k8s.io i list
```

### 安装 crictl 命令


```properties
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock config --list
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock pull 172.17.0.15:85/dev/ddap-mqtt-ws-service:202507292128
# 镜像相关操作
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock image list
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock rmi ~
# 容器相关操作
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock ps
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock rm ~
# Pod相关操作
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock pods list
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock rmp ~
# 内存信息
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock stats
# 获取日志：
crictl --runtime-endpoint=unix:///run/k0s/containerd.sock logs <container-id>
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





























