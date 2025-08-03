# Ceph-deploy集群安装

参考：https://blog.csdn.net/2401_83784772/article/details/140314768

> 注意：
>
> - 官方不推荐 ceph-deploy 工具安装（已废弃），**不支持 Nautilus 以上版本**，也不支持 **RHEL8, CentOS 8**
> - 官方推荐采用 [Cephadm](https://docs.ceph.com/en/latest/cephadm/install/#cephadm-deploying-new-cluster) 工具安装docker版本，该工具仅支持 **Octopus** 以上版本，不支持 **Nautilus** 及以下版本。
> - 官方推荐采用 [Rook](https://rook.io/) 工具安装 k8s 版本，该工具仅支持 **Nautilus** 以上版本。
>
> 详见 https://docs.ceph.com/en/latest/install/
>
> ceph-deploy is not actively maintained. It is not tested on versions of Ceph newer than Nautilus. It does not support RHEL8, CentOS 8, or newer operating systems.

### 本次安装ceph version： 14.2.22（nautilus）

| Name                                                         | Initial release | Latest                                                       | End of life    |
| ------------------------------------------------------------ | --------------- | ------------------------------------------------------------ | -------------- |
| [Squid](https://docs.ceph.com/en/latest/releases/squid)      | **2024-09-26**  | [19.2.2](https://docs.ceph.com/en/latest/releases/squid#v19-2-2-squid) | 2026-09-19     |
| [Reef](https://docs.ceph.com/en/latest/releases/reef)        | 2023-08-07      | [18.2.7](https://docs.ceph.com/en/latest/releases/reef#v18-2-7-reef) | 2025-08-01     |
| [Quincy](https://docs.ceph.com/en/latest/releases/quincy)    | 2022-04-19      | [17.2.9](https://docs.ceph.com/en/latest/releases/quincy#v17-2-9-quincy) | 2025-01-13     |
| [Pacific](https://docs.ceph.com/en/latest/releases/pacific)  | 2021-03-31      | [16.2.15](https://docs.ceph.com/en/latest/releases/pacific#v16-2-15-pacific) | 2024-03-04     |
| [Octopus](https://docs.ceph.com/en/latest/releases/octopus)  | 2020-03-23      | [15.2.17](https://docs.ceph.com/en/latest/releases/octopus#v15-2-17-octopus) | 2022-08-09     |
| [Nautilus](https://docs.ceph.com/en/latest/releases/nautilus) | **2019-03-19**  | [14.2.22](https://docs.ceph.com/en/latest/releases/nautilus#v14-2-22-nautilus) | **2021-06-30** |

# ■■■ 准备环境

|  主机名    |   Public网络   |   Cluster网络   |  角色   |
| ---- | ---- | ---- | ---- |
|   admin   |   20.0.0.10   |      |   admin（管理节点负责集群整体部署）、client   |
| node01 | 172.17.0.40 | 172.17.10.40 | rgw、mon、mgr、osd（/dev/sdb、/dev/sdc、/dev/sdd） |
| node02 | 172.17.0.41 | 172.17.10.41 | rgw、mon、mgr、osd（/dev/sdb、/dev/sdc、/dev/sdd） |
| node03 | 172.17.0.42 | 172.17.10.42 | rgw、mon、mgr、osd（/dev/sdb、/dev/sdc、/dev/sdd） |
| client | 20.0.0.50 |  |  client |

### ■■ 关闭selinux与防火墙

```properties
systemctl disable --now firewalld
systemctl stop firewalld && systemctl disable firewalld
iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat
iptables -P FORWARD ACCEPT

setenforce 0
sed -i 's/enforcing/disabled/' /etc/selinux/config
vi /etc/selinux/config
SELINUX=disabled
```




### ■■ 设置主机名

```properties
hostnamectl set-hostname node01
hostnamectl set-hostname node02
hostnamectl set-hostname node03
---------------------------------
cat /etc/hostname
```

### ■■ 配置host解析

```properties
cat >> /etc/hosts << EOF
172.17.0.40 node01
172.17.0.41 node02
172.17.0.42 node03
EOF
```
### ■■ 安装常用软件和依赖包

```properties
# wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum -y install epel-release
yum -y install yum-plugin-priorities yum-utils ntpdate python-setuptools python-pip gcc gcc-c++ autoconf libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel libxml2 libxml2-devel zlib zlib-devel glibc glibc-devel glib2 glib2-devel bzip2 bzip2-devel zip unzip ncurses ncurses-devel curl curl-devel e2fsprogs e2fsprogs-devel krb5-devel libidn libidn-devel openssl openssh openssl-devel nss_ldap openldap openldap-devel openldap-clients openldap-servers libxslt-devel libevent-devel ntp libtool-ltdl bison libtool vim-enhanced python wget lsof iptraf strace lrzsz kernel-devel kernel-headers pam-devel tcl tk cmake ncurses-devel bison setuptool popt-devel net-snmp screen perl-devel pcre-devel net-snmp screen tcpdump rsync sysstat man iptables sudo libconfig git bind-utils tmux elinks numactl iftop bwm-ng net-tools expect snappy leveldb gdisk python-argparse gperftools-libs conntrack ipset jq libseccomp socat chrony sshpass
```

# ■ 配置时间同步

```properties
systemctl enable --now chronyd
systemctl status chronyd
timedatectl set-ntp true					#开启 NTP
timedatectl set-timezone Asia/Shanghai		#设置时区
chronyc -a makestep							#强制同步下系统时钟
timedatectl status							#查看时间同步状态
chronyc sources -v							#查看 ntp 源服务器信息
timedatectl set-local-rtc 0					#将当前的UTC时间写入硬件时钟
 
#重启依赖于系统时间的服务
systemctl restart rsyslog 
systemctl restart crond
 
#关闭无关服务
systemctl stop postfix && systemctl disable postfix
```

### ■■ 配置Ceph yum源

```properties
wget https://download.ceph.com/rpm-nautilus/el7/noarch/ceph-release-1-1.el7.noarch.rpm --no-check-certificate
rpm -ivh ceph-release-1-1.el7.noarch.rpm --force
```

### ■■安装Ceph软件包

```properties
#若阿里云在线源安装较慢，可使用命令切换为清华大学在线源 vim /etc/yum.repos.d/ceph.repo
sed -i 's#download.ceph.com#mirrors.tuna.tsinghua.edu.cn/ceph#' /etc/yum.repos.d/ceph.repo
yum install -y ceph-mon ceph-radosgw ceph-mds ceph-mgr ceph-osd ceph-common ceph
```

###  ■■ 创建一个Ceph工作目录

**所有节点虚拟机都创建一个 Ceph 工作目录，后续的工作都在该目录下进行**

```properties
mkdir -p /etc/ceph
```

###  ■■ 配置集群网络，添加一个新网卡

在VMware 虚拟机设置里添加新的【**网络适配器**】，如下所示新的**ens224** 网卡

```properties
[root@localhost network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:af:3a:54 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.40/24 brd 172.17.0.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::8b90:ab34:157b:7661/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: ens224: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:af:aa:a3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.149/24 brd 172.17.0.255 scope global noprefixroute dynamic ens224
       valid_lft 79321sec preferred_lft 79321sec
    inet6 fe80::bcd:64a1:836c:4d43/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

### ■■配置新网卡

```properties
cd /etc/sysconfig/network-scripts/
cp ifcfg-ens192 ifcfg-ens224
vim ifcfg-ens224
```

- **BOOTPROTO  改为  static**
- **将 ens192 改为 ens224**
- **删除 UUID，DNS1，GATEWAY**
- **IPADDR改为 172.17.10.40**

```properties
TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="ens224"
DEVICE="ens224"
ONBOOT="yes"
IPADDR="172.17.10.40"
PREFIX="24"
IPV6_PRIVACY="no"
```

- 重启生效

```properties
systemctl restart network
# 查看设置是否成功
ip a
```

### ■■重启所有节点

```properties
reboot
```



---


# ■■■  管理节点安装ceph-deploy工具

> 注意：最新版ceph官方说明 ceph-deploy 部署已废弃

#### ■■ 172.17.0.40 作为管理节点

```properties
cd /etc/ceph
yum install -y ceph-deploy
ceph-deploy --version
----------------------------
2.0.1
```

#### ■■ 配置ssh免密登录其他节点

```properties
ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
cat ~/.ssh/id_rsa
sshpass -p 'Abcd.1234' ssh-copy-id -o StrictHostKeyChecking=no root@node02
sshpass -p 'Abcd.1234' ssh-copy-id -o StrictHostKeyChecking=no root@node03
cat ~/.ssh/known_hosts
```

#### ■■ 生成初始配置

```properties
ceph-deploy new --public-network 172.17.0.0/24 --cluster-network 172.17.10.0/24 node01 node02 node03
```

生成如下文件：

- **ceph.conf：ceph的配置文件**
- **ceph-deploy-ceph.log：monitor的日志**
- **ceph.mon.keyring：monitor的秘钥环文件**

```properties
[root@node01 ceph]# ll
total 16
-rw-r--r-- 1 root root  299 Jul 12 16:22 ceph.conf
-rw-r--r-- 1 root root 6690 Jul 12 16:22 ceph-deploy-ceph.log
-rw------- 1 root root   73 Jul 12 16:22 ceph.mon.keyring
```

**cat ceph.conf**

```properties
[global]
fsid = f89998b5-a407-49c8-a43f-dd5eb280d700
public_network = 172.17.0.0/24
cluster_network = 172.17.10.0/24
mon_initial_members = node01, node02, node03
mon_host = 172.17.0.40,172.17.0.41,172.17.0.42
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx
```



---

# ■■■  部署MON监控节点

#### ■■ 初始化mon节点

```properties
ceph-deploy mon create node01 node02 node03			
# 创建mon节点，由于monitor使用Paxos算法，其高可用集群节点数量要求为大于等于3的奇数台，查看
ps -ef |grep ceph-mon
netstat -tunlp|grep ceph-mon
 
ceph-deploy --overwrite-conf mon create-initial		
# 配置初始化mon节点，并向所有节点同步配置
# --overwrite-conf 参数用于表示强制覆盖配置文件
 
#可选操作，向node01节点收集所有密钥，上面步骤已经生成（跳过）
ceph-deploy gatherkeys node01						
```
- **命令执行成功后会在所有节点 /etc/ceph 下生成配置文件**

```properties
ls /etc/ceph
-------------------------------------------------------------
ceph.bootstrap-mds.keyring			#引导启动 mds 的密钥文件
ceph.bootstrap-mgr.keyring			#引导启动 mgr 的密钥文件
ceph.bootstrap-osd.keyring			#引导启动 osd 的密钥文件
ceph.bootstrap-rgw.keyring			#引导启动 rgw 的密钥文件
ceph.client.admin.keyring			#ceph客户端和管理端通信的认证密钥，拥有ceph集群的所有权限
ceph.conf
ceph-deploy-ceph.log
ceph.mon.keyring
```
#### ■■ 使其它节点可以管理Ceph群集

```properties
cd /etc/ceph
ceph-deploy admin node01 node02 node03			
# 本质就是把 ceph.client.admin.keyring 集群管理认证文件拷贝到各个节点
# 所有节点都可查看集群状态
ceph -s             
```

#### ■■  在三个 node节点上查看自动开启的 mon 进程

```properties
ps aux | grep ceph
----------------------------
ceph        8461  0.0  0.2 505036 35072 ?        Ssl  16:29   0:00 /usr/bin/ceph-mon -f --cluster ceph --id node01 --setuser ceph --setgroup ceph
```

- **在管理节点查看 Ceph 集群状态的mon部分**

```properties
cd /etc/ceph
ceph -s
-------------------------------------------------------
  cluster:
    id:     f89998b5-a407-49c8-a43f-dd5eb280d700
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
 
  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 17m)
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

- **查看 mon 集群选举的情况**
```properties
ceph quorum_status --format json-pretty | grep leader
-------------------------------------------------------
    "quorum_leader_name": "node01",
```
- **扩容 mon 节点（跳过）**
```properties
ceph-deploy mon add <节点名称>-
```

- **每个节点增加了1个mon 进程service**

```properties
[root@node01 ceph]# ps -ef |grep ceph-mon
ceph        8461       1  0 16:29 ?        00:00:30 /usr/bin/ceph-mon -f --cluster ceph --id node01 --setuser ceph --setgroup ceph
```

- **cat /usr/lib/systemd/system/ceph-mon@.service**

```properties
# systemctl status ceph-mon
systemctl is-enabled ceph-mon@node01.service
# 查看启动命令：CLUSTER=ceph
ExecStart=/usr/bin/ceph-mon -f --cluster ${CLUSTER} --id %i --setuser ceph --setgroup ceph
```





---

# ■■■  部署osd存储节点

#### ■■ 3台node各添加三个硬盘

在VMware 虚拟机设置里添加新的【**硬盘**】

#### ■■查看3台node新硬盘

** 注意：主机添加完硬盘后不要分区，直接使用**

```properties
# 如果lsblk 查看不到新硬盘，则在线刷新新硬盘
echo "- - -" > /sys/class/scsi_host/host0/scan
echo "- - -" > /sys/class/scsi_host/host1/scan
echo "- - -" > /sys/class/scsi_host/host2/scan          
#查看新硬盘，sdb sdc sdd
lsblk
----------------------------------------------------
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda               8:0    0  300G  0 disk 
├─sda1            8:1    0  500M  0 part /boot
└─sda2            8:2    0  299G  0 part 
  └─centos-root 253:0    0  299G  0 lvm  /
sdb               8:16   0  300G  0 disk 
sdc               8:32   0  300G  0 disk 
sdd               8:48   0  300G  0 disk 
sr0              11:0    1 1024M  0 rom 
```
#### ■■ 添加osd存储节点

- **在172.17.0.40 管理节点上添加**

```properties
ceph-deploy --overwrite-conf osd create node01 --data /dev/sdb
ceph-deploy --overwrite-conf osd create node02 --data /dev/sdb
ceph-deploy --overwrite-conf osd create node03 --data /dev/sdb
 
ceph-deploy --overwrite-conf osd create node01 --data /dev/sdc
ceph-deploy --overwrite-conf osd create node02 --data /dev/sdc
ceph-deploy --overwrite-conf osd create node03 --data /dev/sdc
 
ceph-deploy --overwrite-conf osd create node01 --data /dev/sdd
ceph-deploy --overwrite-conf osd create node02 --data /dev/sdd
ceph-deploy --overwrite-conf osd create node03 --data /dev/sdd

lsblk
------------------------------------------------------------------
sdb    8:16   0  300G  0 disk 
└─ceph--f5a6fc78-osd--block--856aa75b 253:1    0  300G  0 lvm  
sdc    8:32   0  300G  0 disk 
└─ceph--697b57e9-osd--block--f71d1161 253:2    0  300G  0 lvm  
sdd    8:48   0  300G  0 disk 
└─ceph--0b198d84-osd--block--d62cb379 253:3    0  300G  0 lvm
```
#### ■■ 查看ceph集群状态的osd部分为9 up即配置成功 

- **osd: 9 osds: 9 up (since 13m), 9 in (since 13m)**

```properties
[root@node01 ceph]# ceph -s
  cluster:
    id:     f89998b5-a407-49c8-a43f-dd5eb280d700
    health: HEALTH_WARN
            no active mgr
            mons are allowing insecure global_id reclaim
 
  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 45m)
    mgr: no daemons active
    osd: 9 osds: 9 up (since 36s), 9 in (since 36s)
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

- **每个节点增加了3个osd进程service**

```properties
[root@node01 ceph]# ps -ef |grep ceph
ceph       11256       1  0 17:10 ?        00:00:01 /usr/bin/ceph-osd -f --cluster ceph --id 0 --setuser ceph --setgroup ceph
ceph       11814       1  0 17:11 ?        00:00:01 /usr/bin/ceph-osd -f --cluster ceph --id 3 --setuser ceph --setgroup ceph
ceph       12344       1  0 17:12 ?        00:00:01 /usr/bin/ceph-osd -f --cluster ceph --id 6 --setuser ceph --setgroup ceph
```
- **cat /usr/lib/systemd/system/ceph-osd@.service**

```properties
# systemctl status ceph-osd
systemctl is-enabled ceph-osd@0.service
systemctl is-enabled ceph-osd@3.service  
systemctl is-enabled ceph-osd@6.service

# 查看启动命令：CLUSTER=ceph
ExecStart=/usr/bin/ceph-osd -f --cluster ${CLUSTER} --id %i --setuser ceph --setgroup ceph
ExecStartPre=/usr/lib/ceph/ceph-osd-prestart.sh --cluster ${CLUSTER} --id %i
```





---

# ■■■ 部署mgr管理节点

​	**ceph-mgr 守护进程** **以Active/Standby 模式运行**，可确保在Active节点或其 ceph-mgr 守护进程故障时，其中的一个 **Standby** 实例可以在不中断服务的情况下接管其任务。根据官方的架构原则，mgr至少要有两个节点来进行工作。

```properties
cd /etc/ceph
ceph-deploy mgr create node01 node02 node03
```
- **node01节点增加了mgr进程，其他节点没启动**

```properties
[root@node01 ceph]# ps -ef |grep ceph
ceph       13271       1  7 17:25 ?        00:00:01 /usr/bin/ceph-mgr -f --cluster ceph --id node01 --setuser ceph --setgroup ceph
```
- **cat /usr/lib/systemd/system/ceph-mgr@.service**

```properties
# systemctl status ceph-mgr
systemctl is-enabled ceph-mgr@node01.service
# 查看启动命令：CLUSTER=ceph
ExecStart=/usr/bin/ceph-mgr -f --cluster ${CLUSTER} --id %i --setuser ceph --setgroup ceph
```


#### ■■ 查看 osd 状态（需部署 mgr 后才能执行）

```properties
[root@node01 ceph]# ceph osd status
+----+--------+-------+-------+--------+---------+--------+---------+-----------+
| id |  host  |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
+----+--------+-------+-------+--------+---------+--------+---------+-----------+
| 0  | node01 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 1  | node02 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 2  | node03 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 3  | node01 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 4  | node02 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 5  | node03 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 6  | node01 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 7  | node02 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
| 8  | node03 | 1028M |  298G |    0   |     0   |    0   |     0   | exists,up |
+----+--------+-------+-------+--------+---------+--------+---------+-----------+
```

#### ■■ 查看 osd 容量（需部署 mgr 后才能执行）

```properties
[root@node01 ceph]# ceph osd df
ID CLASS WEIGHT  REWEIGHT SIZE    RAW USE DATA    OMAP META  AVAIL   %USE VAR  PGS STATUS 
 0   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 3   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 6   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 1   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 4   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 7   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 2   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 5   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
 8   hdd 0.29300  1.00000 300 GiB 1.0 GiB 4.8 MiB  0 B 1 GiB 299 GiB 0.33 1.00   0     up 
                    TOTAL 2.6 TiB 9.0 GiB  43 MiB  0 B 9 GiB 2.6 TiB 0.33                 
MIN/MAX VAR: 1.00/1.00  STDDEV: 0
```

```properties
[root@node01 system]# ceph osd tree
ID CLASS WEIGHT  TYPE NAME       STATUS REWEIGHT PRI-AFF 
-1       2.63699 root default                            
-3       0.87900     host node01                         
 0   hdd 0.29300         osd.0       up  1.00000 1.00000 
 3   hdd 0.29300         osd.3       up  1.00000 1.00000 
 6   hdd 0.29300         osd.6       up  1.00000 1.00000 
-5       0.87900     host node02                         
 1   hdd 0.29300         osd.1       up  1.00000 1.00000 
 4   hdd 0.29300         osd.4       up  1.00000 1.00000 
 7   hdd 0.29300         osd.7       up  1.00000 1.00000 
-7       0.87900     host node03                         
 2   hdd 0.29300         osd.2       up  1.00000 1.00000 
 5   hdd 0.29300         osd.5       up  1.00000 1.00000 
 8   hdd 0.29300         osd.8       up  1.00000 1.00000 
```

#### ■■ 查看ceph集群状态的mgr 部分

-  **mgr: node01(active, since 2m), standbys: node03, node02**
-  **active 为 node01，standbys为  node03, node02**
-  **health: HEALTH_WARN**

```properties
[root@node01 ceph]# ceph -s
  cluster:
    id:     f89998b5-a407-49c8-a43f-dd5eb280d700
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
 
  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 58m)
    mgr: node01(active, since 2m), standbys: node03, node02
    osd: 9 osds: 9 up (since 13m), 9 in (since 13m)
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   9.0 GiB used, 2.6 TiB / 2.6 TiB avail
    pgs:
```

#### ■■ 解决HEALTH_WARN问题

- **禁用不安全模式**

```properties
ceph config set mon auth_allow_insecure_global_id_reclaim false
# 重新查看集群状态
ceph -s | grep health
ceph health detail
# 强制创建unkown pg，参考命令：ceph osd force-create-pg {pgid}
# 注：如需批量创建unkown pg，则参考命令如下：
ceph pg dump_stuck inactive
ceph pg dump_stuck inactive | awk '{if (NR>2){print $1}}'
for i in `ceph pg dump_stuck inactive | awk '{if (NR>2){print $1}}'`;do ceph osd force-create-pg $i;done
```





---

# ■■■部署mgr-dashboard

#### ■■ 安装ceph-mgr-dashboard

- **在Active节点上设置：172.17.0.40（node01）**
- **实际上在同一个管理进程上开放8000端口**

```properties
yum install -y ceph-mgr-dashboard
ceph mgr module ls | grep dashboard
-----------------------------------------------
            "name": "dashboard",
```

#### ■■ 配置监控模块

- **在Active节点上设置：172.17.0.40（node01）**

```properties
# 开启 dashboard 模块
ceph mgr module enable dashboard --force
 
# 禁用 dashboard 的 ssl 功能
ceph config set mgr mgr/dashboard/ssl false
 
# 配置 dashboard 监听的地址和端口
ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
ceph config set mgr mgr/dashboard/server_port 8000
 
# 重启 dashboard
ceph mgr module disable dashboard
ceph mgr module enable dashboard --force
 
# 确认访问 dashboard 的 url
ceph mgr services
----------------------------------------------------
{
    "dashboard": "http://node01:8000/"
}
```

#### ■■ 设置账户密码

- **在Active节点上设置：172.17.0.40（node01）**

```properties
echo "12345678" > dashboard_passwd.txt
ceph dashboard set-login-credentials admin -i dashboard_passwd.txt
```

#### ■■ 重置密码：

vi dashboard_password.json

```properties
{"username": "admin", "password": "12345678", "roles": ["administrator"]}
```

```properties
ceph dashboard ac-user-set-password admin -i dashboard_password.json
```


#### ■■ 验证

**浏览器访问：http://172.17.0.40:8000 ，账号密码为 admin/12345678**

```properties
ps -ef |grep ceph-mgr
netstat -tunlp|grep ceph-mgr
```





---

# ■■■  部署RADOS Gateway

- **注意服务名为 ceph-radosgw@rgw.node1，默认端口号为 7480**

```properties
 ceph-deploy rgw create node01 node02 node03
 # http://node01:7480，可以看到输出一个XML
```

**初始化完成 radosgw 之后，会初始化默认的存储池如下：**

**名称以 default.rgw.* 为前缀和 .rgw.root的存储池**

```properties
[root@node01 ceph]# ceph osd pool ls
.rgw.root
default.rgw.control
default.rgw.meta
default.rgw.log

# 验证radosgw服务进程
[root@node01 ceph]# ps -ef|grep radosgw
ceph      251977       1  0 18:09 ?        00:00:02 /usr/bin/radosgw -f --cluster ceph --name client.rgw.node01 --setuser ceph --setgroup ceph

```

### ■■ 查看默认 radosgw 的存储池信息：
#### radosgw-admin zone get --rgw-zone=default --rgw-zonegroup=default

```properties
{
    "id": "1b3dea80-420a-41aa-aade-5fa096dfbd05",
    "name": "default",
    "domain_root": "default.rgw.meta:root",
    "control_pool": "default.rgw.control",
    "gc_pool": "default.rgw.log:gc",
    "lc_pool": "default.rgw.log:lc",
    "log_pool": "default.rgw.log",
    "intent_log_pool": "default.rgw.log:intent",
    "usage_log_pool": "default.rgw.log:usage",
    "reshard_pool": "default.rgw.log:reshard",
    "user_keys_pool": "default.rgw.meta:users.keys",
    "user_email_pool": "default.rgw.meta:users.email",
    "user_swift_pool": "default.rgw.meta:users.swift",
    "user_uid_pool": "default.rgw.meta:users.uid",
    "otp_pool": "default.rgw.otp",
    "system_key": {
        "access_key": "",
        "secret_key": ""
    },
    "placement_pools": [
        {
            "key": "default-placement",
            "val": {
                "index_pool": "default.rgw.buckets.index",
                "storage_classes": {
                    "STANDARD": {
                        "data_pool": "default.rgw.buckets.data"
                    }
                },
                "data_extra_pool": "default.rgw.buckets.non-ec",
                "index_type": 0
            }
        }
    ],
    "metadata_heap": "",
    "realm_id": ""
}
```

- rgw.root： 包含 realm(领域信息)，比如 zone 和 zonegroup
- default.rgw.log： 存储日志信息，用于记录各种 log 信息。
- default.rgw.control： 系统控制池，在有数据更新时，通知其它 RGW 更新缓存。
- default.rgw.meta： 元数据存储池，通过不同的名称空间分别存储不同的 rados 对象，这些名称空间包括⽤⼾UID 及其 bucket 映射信息的名称空间 users.uid、⽤⼾的密钥名称空间users.keys、⽤⼾的 email 名称空间 users.email、⽤⼾的 subuser 的名称空间 users.swift，以及 bucket 的名称空间 root 等。
- default.rgw.buckets.index： 存放 bucket 到 object 的索引信息。
- default.rgw.buckets.data： 存放对象的数据。
- default.rgw.buckets.non-ec： 数据的额外信息存储池

- default.rgw.users.uid： 存放用户信息的存储池。

- default.rgw.data.root： 存放 bucket 的元数据，结构体对应 RGWBucketInfo，比如存放桶名、桶 ID、data_pool 等。

### ■■ 查看对象存储池的存储策略、副本数量、pgp和pg的数量

```properties
[root@node01 ceph]# ceph osd pool get default.rgw.meta crush_rule
crush_rule: replicated_rule
[root@node01 ceph]# ceph osd pool get default.rgw.meta size
size: 3
[root@node01 ceph]# ceph osd pool get default.rgw.meta pgp_num
pgp_num: 32
[root@node01 ceph]# ceph osd pool get default.rgw.meta pg_num
pg_num: 32
```

## ■■■ 端口 7480改为80

```properties
vim /etc/ceph/ceph.conf 
# 输出-------------------------------
[client.rgw.node01]
rgw_frontends = "civetweb port=172.17.0.40:7480"
[client.rgw.node02]
rgw_frontends = "civetweb port=172.17.0.41:7480"
[client.rgw.node03]
rgw_frontends = "civetweb port=172.17.0.42:7480"
 
# 重启 ceph-radosgw
systemctl restart ceph-radosgw@rgw.node01
systemctl status ceph-radosgw@rgw.node01 
 
# 查看端口80
netstat -tunlp|grep radosgw
```





---

# ■■■  Dashboard中启用RGW

###  ■■■■  创建rgw dashboard用户

```properties
echo "cx-admin" > dashboard_passwd2.txt
ceph dashboard ac-user-create cx-user -i dashboard_passwd2.txt administrator

radosgw-admin user create --uid=rgw-dashboard --display-name=rgw-dashboard --system
# 查看 access_key 和 secret_key的值
radosgw-admin user info --uid=rgw-dashboard
radosgw-admin metadata list user 
#----------------------------------------------------
"access_key": "X2SENFUWAHIT73RBZEIM",
"secret_key": "gn6R6SpMR97R9iC6wucPYhaZKRoSw7B1hBPJoYYY"
#----------------------------------------------------
# 设置access_key 和 secret_key的值
echo "X2SENFUWAHIT73RBZEIM" > rgw-api-access-key.txt
echo "gn6R6SpMR97R9iC6wucPYhaZKRoSw7B1hBPJoYYY" > rgw-api-secret-key.txt
ceph dashboard set-rgw-api-access-key -i rgw-api-access-key.txt
ceph dashboard set-rgw-api-secret-key -i rgw-api-secret-key.txt
#即可在Dashboard 查看buckets
```


```properties
radosgw-admin bucket stats --bucket=ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6
radosgw-admin zone list
radosgw-admin metadata list user
radosgw-admin user info --uid=obc-default-ceph-delete-bucket-8061dd3d-1501-4e4a-b430-b082aa81dbdf
----------------------------------------------
  "access_key": "U7M250OYM5BR3NCZSZPX",
  "secret_key": "yLHtcMH1IVz7Pdfr39iCM0Qnw1XCcA2Bslp5q7tC",
```


## ■■■ 迁移桶数据

#### vim /root/.s3cfg_k8s

```properties
[default]
access_key = U7M250OYM5BR3NCZSZPX
secret_key = yLHtcMH1IVz7Pdfr39iCM0Qnw1XCcA2Bslp5q7tC
host_base = 172.17.0.222:30139
host_bucket = 172.17.0.222:30139
use_https = False
human_readable_sizes = True
website_index = index.html
```

### 查看桶

```properties
s3cmd ls --config=/root/.s3cfg_k8s
s3cmd ls s3://ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6 --config=/root/.s3cfg_k8s
#------------------------------------------------------------------------------------
                    DIR  s3://ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6//
                    DIR  s3://ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6/root/
2025-04-12 08:38    11   s3://ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6/rookObj
```
## ■■■ 导出

```properties
mkdir -p /root/data/tmp
s3cmd sync s3://ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6// /root/data/tmp/ --config=/root/.s3cfg_k8s --recursive --stats

mkdir -p /root/data/root
s3cmd sync s3://ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6/root/ /root/data/root/ --config=/root/.s3cfg_k8s --recursive --stats

mkdir -p /root/data/rookObj
s3cmd sync s3://ceph-bkt-c255a143-82cd-4037-908f-14c2c1aef7e6/rookObj /root/data/rookObj/ --config=/root/.s3cfg_k8s --recursive --stats
```
## ■■■ 导入

#### vim /root/.s3cfg

```properties
[default]
access_key = X2SENFUWAHIT73RBZEIM
secret_key = gn6R6SpMR97R9iC6wucPYhaZKRoSw7B1hBPJoYYY
host_base = 172.17.0.40:7480
host_bucket = 172.17.0.40:7480
use_https = False
human_readable_sizes = True
website_index = index.html
```

- **装完nginx+ keepalived可修改IP和PORT**

### 导出命令

```properties
s3cmd ls 

s3cmd sync /root/data/tmp/ s3://cx-food// --config=/root/.s3cfg --acl-private --no-check-md5 --multipart-chunk-size-mb=512 --recursive --stats
s3cmd sync /root/data/root/ s3://cx-food/root/ --config=/root/.s3cfg --acl-private --no-check-md5 --multipart-chunk-size-mb=512 --recursive --stats
s3cmd sync /root/data/rookObj/rookObj s3://cx-food/rookObj --config=/root/.s3cfg --recursive --stats
```

## ■■■ 查看

```properties
s3cmd ls s3://cx-food//
radosgw-admin metadata list bucket
radosgw-admin metadata list bucket.instance
radosgw-admin metadata list user
radosgw-admin metadata get bucket:${BUCKET_NAME}
```

------



#  部署 nginx高可用

#### mkdir -p /k8s/nginx/conf/

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
        upstream ceph_tcp {
            hash $remote_addr consistent;
            server 172.17.0.40:7480 max_fails=3 fail_timeout=30s;
            server 172.17.0.41:7480 max_fails=3 fail_timeout=30s;
            server 172.17.0.42:7480 max_fails=3 fail_timeout=30s;
        }

        server {
            listen 80;
            proxy_connect_timeout 2s;
            proxy_timeout 120;
            proxy_pass ceph_tcp;
        }
    }
EOF
```
#### 配置 systemd unit 文件，启动服务

```properties
cat << EOF | tee /etc/systemd/system/kube-nginx.service
[Unit]
Description=emqx nginx proxy
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

------

#  部署 keepalived

### 入口3节点安装

```properties
yum install -y gcc openssl-devel popt-devel
mkdir -p /k8s/keepalived && cd /k8s/keepalived
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

### 创建check_nginx.sh

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

### 【40】节点

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
   router_id ceph@172.17.0.40
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 77
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.77/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```

### 【41】节点

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
   router_id ceph@172.17.0.41
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    virtual_router_id 77
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.77/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```
### 【42】节点

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
   router_id ceph@172.17.0.42
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    virtual_router_id 77
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.77/24
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
```

#### 查看日志文件
```properties
tail -f -n 500 /var/log/messages
```

------

## Ceph对外IP端口

```properties
ip a
172.17.0.77:80
```

#### 修改IP和PORT：vim /root/.s3cfg

```properties
[default]
access_key = X2SENFUWAHIT73RBZEIM
secret_key = gn6R6SpMR97R9iC6wucPYhaZKRoSw7B1hBPJoYYY
host_base = 172.17.0.77
host_bucket = 172.17.0.77
use_https = False
human_readable_sizes = True
website_index = index.html
```

### 验证

```properties
s3cmd ls
```





---

# ■ 创建CephFS(生产禁用)

- **集群创建完后， 默认没有文件系统，** 
- 我们创建一个 Cephfs 可以支持对外访问的文件系统。 
```properties
 ceph-deploy --overwrite-conf mds create node01 node02 node03
```
#### ■■创建两个存储池, 执行两条命令：

```properties
ceph osd pool create cephfs_data 512
ceph osd pool create cephfs_metadata 64
```

- 少于 5 个 OSD 可把 pg_num 设置为 128  

- **OSD 数量在 5 到 10 ，可以设置 pg_num 为 512**  

- OSD 数量在 10 到 50 ，可以设置 pg_num 为 4096  

- OSD 数量大于 50 ，需要计算 pg_num 的值  

####  ■■查看存储池
```properties
ceph osd lspools
```

####  ■■创建fs, 名称为fs_test:

```properties
ceph fs new fs_test cephfs_metadata cephfs_data
```

#### ■■状态查看， 以下信息代表正常：

```properties
[root@CENTOS7-1 mgr-dashboard]# ceph fs ls
name: fs_test, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

```properties
[root@CENTOS7-1 ceph-cluster]# ceph mds stat
fs_test-1/1/1 up  {0=CENTOS7-2=up:active}, 2 up:standby
```

- 附： 如果创建错误， 需要删除， 执行： 

```properties
ceph fs rm fs_test --yes-i-really-mean-it 
ceph osd pool delete cephfs_data cephfs_data --yes-i-really-really-mean-it
```

- 确保在ceph.conf中开启以下配置：

```properties
[mon] mon allow pool delete = true
```

#### ■■采用fuse挂载

- **先确定 ceph-fuse 命令能执行， 如果没有， 则安装：**  

```properties
yum -y install ceph-fuse
```

#### ■■创建挂载目录

```properties
mkdir -p /usr/local/cephfs_directory
```

#### ■■挂载cephfs

```properties
ceph-fuse -k /etc/ceph/ceph.client.admin.keyring -m 192.168.88.161:6789 /usr/local/cephfs_directory
```

#### ■■查看磁盘挂载信息

```properties
df -h
```


