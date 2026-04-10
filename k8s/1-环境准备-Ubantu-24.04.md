# k8s环境准备-Ubantu-24.04

本文是直接下载二进制安装，也可以进入[k8s的github](https://github.com/kubernetes/kubernetes),clone下来编译成二进制安装, 可查看本文件夹下《编译和运行Kubernetes源码》。

## 版本说明

| 组件    | 版本号           |      | 组件           | 版本号       |
| ------- | ---------------- | ---- | -------------- | ------------ |
| ubantu  | 24.04            |      | dashboard      | 1.10.1       |
| k8s     | 1.13.3           |      | metrics-server | 0.3.1        |
| docker  | 18.06.2.ce-3.el7 |      | traefik        | 1.7.7-alpine |
| etcd    | 3.2.24           |      | heketi         | 8.0.0        |
| calico  | 3.3.1            |      | glusterfs      | 4.1          |
| coredns | 1.2.6            |      |                |              |

## 网络规划

| 网络类型 |    地址网段     |
| :------: | :-------------: |
| 物理网段 | 192.168.10.0/24 |
| POD网段  |  172.30.0.0/16  |
| SVC网段  |  10.254.0.0/16  |

## 二进制下载

*   [k8s-v1.13.3-amd64](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.13.md)
*   [etcd-v3.2.24](https://github.com/etcd-io/etcd/releases/tag/v3.2.24)
*   [calico-v3.3.1](https://github.com/projectcalico/calico/releases/tag/v3.3.1)

***

本次采用二进制文件方式部署

不建议使用secureCRT这个ssh软件复制本篇博客内容的命令,因为它的部分版本对包含多条命令的处理结果并不完美,可能很多命令不是预期结果

本文命令里有些是输出,不要乱粘贴输入(虽然也没影响)

本文命令全部是在k8s-m1（192.168.10.110）上执行

本文很多步骤是选择其一,别啥都不看一路往下复制粘贴

如果某些步骤理解不了可以上下内容一起看来理解

本文后面的几个svc用了externalIPs,上生产的话必须用VIP或者LB代替

本文的HA是vip,生产和云上可以用LB和SLB,不过阿里的SLB四层有问题,可以每个node上代理127.0.0.1的某个port分摊在所有apiserver的port上,aws的SLB正常

master节点一定要kube-proxy和calico或者flannel,kube-proxy是维持svc的ip到pod的ip的负载均衡，而你流量想到pod的ip需要calico或者flannel组件的overlay网络下才可以，后续学到APIService和CRD的时候，APIService如果选中了svc，kube-apiserver会把这个APISerivce的请求代理到选中的svc上，最后流量流到svc选中的pod，该pod要处理请求然后回应，这个时候就是kube-apiserver解析svc的名字得到svc的ip，然后kube-proxy定向到pod的ip，calico或者flannel把包发到目标机器上，这个时候如果kube-proxy和calico或者flannel没有那你创建的APISerivce就没用了。本文中的metrics-server就是这样的工作流程，所以建议master也跑pod，不然后续某些CRD用不了

## 主机名

设置永久主机名称，然后重新登录:

```properties
# 将 m7-autocv-gpu01 替换为当前主机名
hostnamectl set-hostname m7-autocv-gpu01 
```

*   设置的主机名保存在 `/etc/hostname` 文件中；

**如果 DNS 不支持解析主机名称**，则需要修改每台机器的 `/etc/hosts` 文件，添加主机名和 IP 的对应关系：

```properties
cat >> /etc/hosts <<EOF
172.27.128.150 m7-autocv-gpu01	m7-autocv-gpu01
172.27.128.149 m7-autocv-gpu02	m7-autocv-gpu02
172.27.128.148 m7-autocv-gpu03	m7-autocv-gpu03
EOF
```

## 无密码 ssh 登录其它节点

```properties
ssh-keygen -t rsa 
# 一路按enter，生成
ssh-copy-id root@192.168.10.111
ssh-copy-id root@192.168.10.112
ssh-copy-id root@192.168.10.120
```

#### 更换国内源

```properties
sudo vim /etc/apt/sources.list.d/ubuntu.sources
# 修改成
URIs: http://mirrors.aliyun.com/ubuntu/
```
#### 添加 apt 代理
```properties
sudo vim /etc/apt/apt.conf.d/95proxies
#-------------------------------------------------
Acquire::http::Proxy "http://10.10.9.252:7897";
Acquire::https::Proxy "http://10.10.9.252:7897";
#-------------------------------------------------
sudo apt update && sudo apt upgrade
```
## 安装依赖包

在每台机器上安装依赖包：ipvs 依赖 ipset；

```properties
sudo apt install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp-dev
```

***

## 二、Kubernetes 安装及配置

### 1、初始化环境

#### 关闭防火墙

```properties
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat
sudo iptables -P FORWARD ACCEPT
```

#### 禁用 AppArmo

```properties
sudo apparmor_status
sudo systemctl status apparmor
sudo systemctl stop apparmor
sudo systemctl disable apparmor
```

#### 永久关闭 swap

```properties
# 临时关闭
sudo swapoff -a && sudo sysctl -w vm.swappiness=0
# 永久关闭
sudo vim /etc/fstab
# 注释了下面这一行
#/swap.img	none	swap	sw	0	0

# 校验
free
```

## 安装时间同步服务器 chrony 

> **注意：**
> **ntpd 和 chronyd 服务互相冲突，二选一**

- 201 服务器，202/203客户端

```properties
sudo apt install chrony
sudo vim /etc/chrony/chrony.conf
# 删除 4 行 pool 行内容, 添加下面的ntp服务器
#--------------------------------------------
server ntp.aliyun.com iburst
server ntp.tencent.com iburst
server ntp.tuna.tsinghua.edu.cn iburst
server cn.pool.ntp.org iburst
server ntp.ntsc.ac.cn iburst

#-- 其他客户端 配置----------------------
server 10.10.20.201 iburst
# 在最后面添加下面内容
allow 0.0.0.0/0
local stratum 10
#--------------------------------------
sudo systemctl daemon-reload
sudo systemctl restart chronyd
sudo systemctl enable --now chrony
# 检查 123 端口
ss -ntul |grep 123
sudo systemctl status chrony
# 检查偏差，应该接近0
sudo chronyc tracking | grep "System time"
# 立即强制同步
sudo chronyc makestep
```

### 配置时间同步chronyd

```properties
# 查看是否关闭 ntpd
systemctl is-enabled ntpd
systemctl is-enabled ntpdate

# 启动 chronyd
systemctl enable --now chronyd
systemctl status chronyd
# 开启 timedatectl NTP
timedatectl set-ntp true
# 设置时区
timedatectl set-timezone Asia/Shanghai	
# 强制同步下系统时钟
chronyc -a makestep
# 查看时间同步状态
timedatectl status
# 查看 ntp 源服务器信息
chronyc sources -v
# 将当前的UTC时间写入硬件时钟
timedatectl set-local-rtc 0					
 
# 重启依赖于系统时间的服务
sudo systemctl restart rsyslog 
sudo systemctl restart crond
 
# 关闭无关服务
sudo systemctl stop postfix && systemctl disable postfix
```

#### 关闭 dnsmasq

linux 系统开启了 dnsmasq 后(如 GUI 环境)，将系统 DNS Server 设置为 127.0.0.1，这会导致 docker 容器无法解析域名，需要关闭它：

```properties
systemctl stop dnsmasq && systemctl disable dnsmasq
```



## 设置 rsyslogd 和 systemd journald

systemd 的 journald 是 Centos 7 缺省的日志记录工具，它记录了所有系统、内核、Service Unit 的日志。

相比 systemd，journald 记录的日志有如下优势：

1.  **可以记录到内存或文件系统；(默认记录到内存，对应的位置为 /run/log/jounal)**
2.  **可以限制占用的磁盘空间、保证磁盘剩余空间；**
3.  **可以限制日志文件大小、保存的时间；**

journald 默认将日志转发给 rsyslog，这会导致日志写了多份，**/var/log/messages 中包含了太多无关日志，不方便后续查看，同时也影响系统性能。**

```properties
# 持久化保存日志的目录
sudo mkdir /var/log/journal 
sudo mkdir /etc/systemd/journald.conf.d
--------------------------------------
# /etc/systemd/journald.conf.d/99-prophet.conf
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间 10G
SystemMaxUse=10G
# 单日志文件最大 200M
SystemMaxFileSize=200M
# 日志保存时间 2 周
MaxRetentionSec=2week
# 不将日志转发到 syslog
ForwardToSyslog=no
---------------------------------------------------------------------------
sudo tee /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
Storage=persistent
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
SystemMaxUse=10G
SystemMaxFileSize=200M
MaxRetentionSec=2week
ForwardToSyslog=no
EOF
# 重启 systemd-journald
sudo systemctl restart systemd-journald
sudo systemctl status systemd-journald
```
---

## 查看这个内核里是否有这个内核模块

```properties
# 查看内核是否最新版本
uname -r
find /lib/modules -name '*nf_conntrack_ipv4*' -type f
# 这是输出 5.4.278-1.el7.elrepo.x86_64下没有 nf_conntrack_ipv4
find /lib/modules -name '*nf_conntrack*' -type f
# 查看此文件是否存在（升级后不存在）
ll /proc/sys/fs/may_detach_mounts
```

# 加载模块

```properties
# 内核 >= 4.19
sudo modprobe nf_conntrack
sudo modprobe ip_vs
sudo modprobe br_netfilter
```
## 所有`node`机器安裝 socat(用于helm端口转发)：

```properties
sudo apt install -y socat
```

## 所有机器 安装 ipvs

（k8s 1.11后使用 ipvs，性能甩 iptables几条街）

```properties
sudo apt install -y ipvsadm ipset sysstat conntrack libseccomp2
```

**所有机器** 选择需要开机加载的内核模块,以下是 ipvs 模式需要加载的模块并设置开机自动加载

```properties
$ sudo sh -c '> /etc/modules-load.d/ipvs.conf'
$ sudo cat /etc/modules-load.d/ipvs.conf
$ module=(
  ip_vs
  ip_vs_lc
  ip_vs_wlc
  ip_vs_rr
  ip_vs_wrr
  ip_vs_lblc
  ip_vs_lblcr
  ip_vs_dh
  ip_vs_sh
  ip_vs_fo
  ip_vs_nq
  ip_vs_sed
  ip_vs_ftp
  )

$ for kernel_module in "${module[@]}"; do
    sudo modinfo -F filename "$kernel_module" 2>/dev/null | grep -qv ERROR && echo "$kernel_module" | sudo tee -a /etc/modules-load.d/ipvs.conf > /dev/null || :
done

sudo systemctl enable --now systemd-modules-load
sudo systemctl restart systemd-modules-load
sudo systemctl status systemd-modules-load

lsmod | grep ip_vs
sudo ipvsadm -L -n
```

上面如果systemctl enable命令报错可以 **systemctl status -l systemd-modules-load** 看看哪个内核模块加载不了，在/etc/modules-load.d/ipvs.conf 里注释掉它，再enable试试



## 所有机器需要设定 /etc/sysctl.d/kubernetes.conf  的系统参数

参照上文，与原文增加了

- net.ipv4.tcp_tw_recycle=0， 内核版本 3.7 开始废弃
- fs.inotify.max_user_instances=8192
- **may_detach_mounts=1，开启，为什么升级后的内核版本4.4.X、 5.4.X 不支持？**
- ll /proc/sys/fs/may_detach_mounts

```properties
# https://github.com/moby/moby/issues/31208 
# ipvsadm -l --timout
# 修复ipvs模式下长连接timeout问题 小于900即可
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.ipv4.tcp_keepalive_time=600
net.ipv4.tcp_keepalive_intvl=30
net.ipv4.tcp_keepalive_probes=10
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1
net.ipv4.neigh.default.gc_stale_time=120
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.default.arp_announce=2
net.ipv4.conf.lo.arp_announce=2
net.ipv4.conf.all.arp_announce=2
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets=5000
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.tcp_synack_retries=2
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-arptables=1
net.netfilter.nf_conntrack_max=2310720
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
EOF

#生效配置文件
sudo sysctl --system
sudo sysctl -p /etc/sysctl.d/kubernetes.conf
```

张馆长结束

***


## ~~检查系统内核和模块是否适合运行 docker (跳过)~~

- **从 Kubernetes 1.24 开始，Docker[不再支持](https://kubernetes.io/blog/2020/12/02/dockershim-faq/)开箱即用地作为容器运行时**

```properties
$ curl https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh > check-config.sh
$ bash ./check-config.sh
```

若docker官方的内核检查脚本建议(RHEL7/CentOS7: User namespaces disabled; add ‘user\_namespace.enable=1’ to boot command line),使用下面命令开启

```properties
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"
# 重启
reboot
```

# ~~Docker安装（跳过）~~

- 从 Kubernetes 1.24 开始，Docker[不再支持](https://kubernetes.io/blog/2020/12/02/dockershim-faq/)开箱即用地作为容器运行时

- 参考[阿里云](https://help.aliyun.com/document_detail/60742.html?spm=5176.11065259.1996646101.searchclickresult.261ebdd3Qy7wLJ)，参考[k8s源码](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.13.md#external-dependencies)对应的版本号再安装

### docker 安装步骤

- https://docs.docker.com/engine/install/ubuntu/

```properties
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

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

#### 手动设置docker命令补全

```properties
yum install -y epel-release bash-completion
cp /usr/share/bash-completion/completions/docker /etc/bash_completion.d/
```

#### 开启Docker服务

```properties
systemctl daemon-reload && systemctl enable docker && systemctl start docker
systemctl status docker
```

#### 安装校验

```properties
#验证是否安装成功
docker info
docker version
```

#### 删除Docker(别做)

```properties
# 删除安装包
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
# 删除目录
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
# 删除源
sudo rm /etc/apt/sources.list.d/docker.sources
sudo rm /etc/apt/keyrings/docker.asc
```

## 创建定时删除docker镜像垃圾文件

*   新建clearDockerNone.sh

```properties
cat << EOF | tee ~/clearDockerNone.sh
#!/bin/bash
docker ps -a | grep "Exited" | awk '{print \$1 }'|xargs docker stop
docker ps -a | grep "Exited" | awk '{print \$1 }'|xargs docker rm
# docker images|grep none|awk '{print $3 }'|xargs docker rmi
docker image prune -a -f
EOF
```

*   新建root用户的cron服务

```properties
# 编辑某个用户的cron服务
$ crontab -e
--------------------------------------------------------
# every Saturday 2:05 excute clear docker none image
5 2 * * 6 /bin/bash ~/clearDockerNone.sh > /dev/null 2>&1
# 列出某个用户cron服务的详细内容 
$ crontab -l
# 设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数,
# 比如说root查看自己的cron设置: crontab -u root -l
$ crontab -u
```

### 配置harbor镜像证书(正式域名证书忽略此步)

```properties
#先下载 harbor镜像证书 ca.crt，再拷贝到指定域名目录下
mkdir -p /etc/docker/certs.d/harbor-sit.jrtzcloud.cn/

# 到 110 机器
cd /etc/docker/certs.d/harbor-sit.jrtzcloud.cn/
scp ca.crt 192.168.10.XXX:/etc/docker/certs.d/harbor-sit.jrtzcloud.cn/
```


# 所有机器安装CNI插件（先跳过）

```properties
# 已经下载忽略（D:\learning\k8s\jrtz\bins\cni-plugins-amd64-v0.6.0）
# wget https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz
mkdir -p /k8s/kubernetes/bin
mkdir -p /k8s/cni/bin
# tar -zxf cni-plugins-amd64-v0.6.0.tgz -C /k8s/cni/bin
# 进入 110 机器
scp /k8s/cni/bin/* 192.168.10.XXX:/k8s/cni/bin
```

#### 添加环境变量

将可执行文件路/k8s/kubernetes/ 添加到 PATH 变量中

```properties
vim /etc/profile
-------------------------
# 在文末尾(最后一行 unset -f pathmunge)添加
PATH=$PATH:/k8s/kubernetes/bin:/k8s/cni/bin
export PATH
-------------------------
# 重新加载生效
source /etc/profile
```

## 所有`node`机器安装Glusterfs客户端（先跳过）

*   每个kubernetes集群的节点需要安装gulsterfs的客户端，

```properties
yum -y install centos-release-gluster41.noarch
yum -y install glusterfs-client
```

*   加载内核模块：每个kubernetes集群的节点运行

```properties
modprobe dm_thin_pool
```

## 创建文件夹

```properties
mkdir -p /k8s/kubernetes/ssl
```

## 修改系统最大进程数

[参考CSDN-1](https://blog.csdn.net/gatieme/article/details/51058797)

[参考CSDN-2](https://blog.csdn.net/qq_35963057/article/details/81489907)

* 查看系统中可创建的进程数实际值(32768)

   `cat /proc/sys/kernel/pid_max`

> 为了与老版本的Unix或者Linux兼容，PID的最大值默认设置位32768(short int 短整型的最大值)。

* 查看当前系统中单进程可打开的文件描述符数目(52706963)

   `cat /proc/sys/fs/file-max`

* 查看内核支持的最大file handle数量，即一个进程最多使用的file handle数(52706963)

   `cat /proc/sys/fs/nr_open`

* `ulimit -Sn` (1024)查看最大打开文件描述符数的soft limit，注意soft limit不能大于hard limit,否则注销后无法正常登录

* `ulimit -Hn` (4096)查看最大打开文件描述符数的hard limit

### 说明

file-max是内核可分配的最大文件数，nr\_open是单个进程可分配的最大文件数，前面ipvs已经设置 `file-max=nr_open=52706963`, 注意事项：

*   所有进程打开的文件描述符数不能超过/proc/sys/fs/file-max

*   单个进程打开的文件描述符数不能超过user limit中nofile的soft limit

*   nofile的soft limit不能超过其hard limit

*   nofile的hard limit不能超过/proc/sys/fs/nr\_open

***

*   查看 `ulimit -a`
*   查看 `cat /etc/security/limits.d/20-nproc.conf`

```properties
*          soft    nproc     4096
root       soft    nproc     unlimited
```

*   修改参数（增加） `vim /etc/security/limits.conf`

```properties
*       soft    nproc   131072
*       hard    nproc   131072
*       soft    nofile  131072
*       hard    nofile  131072
root    soft    nproc   131072
root    hard    nproc   131072
root    soft    nofile  131072
root    hard    nofile  131072
```

## 增加 vm.max\_map\_count=262144(暂时不做，Elastic专用)

```bash
$ sysctl -a |grep vm.max_map_count
----------
vm.max_map_count = 65530
$ echo 'vm.max_map_count=262144' >> /etc/sysctl.conf
$ cat /etc/sysctl.conf
```

***

## 先重启，此时可以关机做个快照，然后重启