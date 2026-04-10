# Glusterfs虚拟机安装手顺

## 初始化

网络要求全部千兆环境，gluster 服务器至少有 **2 块网卡**，1 块网卡绑定供 gluster 使用，剩余一块分配管理网络 IP，用于系统管理。如果有条件购买**万兆交换机**，服务器配置**万兆网卡**，存储性能会更好。网络方面如果安全性要求较高，可以多网卡绑定。
分布式文件系统的时候数据盘一般不需要做 RAID，一般系统盘会做 RAID 1；
如果有raid卡的话，最好用上，raid卡有数据缓存功能，也能提高磁盘的iops，最好的话，用RAID 5；
如果都不做raid的话，也是没问题的，glusterfs也是可以保证数据的安全的。

### 服务规划

其中sda为系统盘，sdd为日志盘（/var/log）,sdb和sdc为数据挂载盘

| 操作系统   | IP             | 主机名        | 硬盘数量（三块）                      | 用途    |
| ---------- | -------------- | ------------- | ------------------------------------- | ------- |
| centos 7.6 | 192.168.10.113 | vmsrv-010-113 | sda:100G  sdb:500G sdc:1024G sdd:100G | gluster |
| centos 7.6 | 192.168.10.122 | vmsrv-010-122 | sda:100G  sdb:500G sdc:1024G sdd:100G | gluster |
| centos 7.6 | 192.168.10.123 | vmsrv-010-143 | sda:100G  sdb:500G sdc:1024G sdd:100G | gluster |
| centos 7.6 | 192.168.10.221 | vmsrv-010-231 | sda:100G  sdb:500G sdc:1024G sdd:100G | gluster |
| centos 7.6 | 192.168.10.124 | vmsrv-010-124 | sda:100G                              | heketi  |

### 初始化系统文件

首先关闭iptables和selinux，配置hosts文件如下（全部glusterfs主机）

```bash
$ cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.10.113 vmsrv-010-113
192.168.10.122 vmsrv-010-122
192.168.10.123 vmsrv-010-123
192.168.10.221 vmsrv-010-221
---------------------------------------参考环境准备文章
# 停止firewalld
$ systemctl stop firewalld.service  
# 禁止firewalld开机自启
$ systemctl disable firewalld.service  
# 关闭SELinux
$ sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config  
$ setenforce 0
$ getenforce
Permissive
#同步时间
$ntpdate time.windows.com   
```

### 加载内核模块

```bash
modprobe dm_thin_pool && modprobe dm_snapshot && modprobe dm_mirror
```

### 日志目录单独挂载盘

```bash
fdisk -l
# 格式化所有数据盘和日志盘
mkfs.xfs  -i size=512 /dev/sdX
# 将日志盘挂载到/var/log
echo '/dev/sdX /var/log xfs defaults 1 2' >> /etc/fstab
#将/etc/fstab的所有内容重新加载
mount -a && mount
df -h
```

### 安装gluterfs源（全部主机）

```bash
$ yum search  centos-release-gluster
=================N/S matched: centos-release-gluster =========================
centos-release-gluster-legacy.noarch : Disable unmaintained Gluster repositories from the CentOS Storage SIG
centos-release-gluster40.x86_64 : Gluster 4.0 (Short Term Stable) packages from the CentOS Storage SIG repository
centos-release-gluster41.noarch : Gluster 4.1 (Long Term Stable) packages from the CentOS Storage SIG repository
centos-release-gluster5.noarch : Gluster 5 packages from the CentOS Storage SIG repository
centos-release-gluster6.noarch : Gluster 6 packages from the CentOS Storage SIG repository
centos-release-gluster7.noarch : Gluster 7 packages from the CentOS Storage SIG repository
```

### 选择 7 版本

```bash
yum  -y  install centos-release-gluster7.noarch
```

### 安装glusterfs（全部主机）

查看glusterfs7源

```
cat /etc/yum.repos.d/CentOS-Gluster-7.repo
```

安装glusterfs7

```bash
yum install glusterfs-server
```

### 查看版本并启动服务(全部主机)

```bash
$ glusterfs -V
glusterfs 7.2
$ systemctl enable glusterd
ln -s '/usr/lib/systemd/system/glusterd.service' '/etc/systemd/system/multi-user.target.wants/glusterd.service'
$ systemctl start glusterd
$ systemctl status glusterd
-----------------------------------
glusterd.service - GlusterFS, a clustered file-system server
   Loaded: loaded (/usr/lib/systemd/system/glusterd.service; enabled)
   Active: active (running) since Fri 2015-11-13 10:16:09 CET; 3s ago
  Process: 25972 ExecStart=/usr/sbin/glusterd -p /var/run/glusterd.pid --log-level $LOG_LEVEL $GLUSTERD_OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 25973 (glusterd)
   CGroup: /system.slice/glusterd.service
           └─25973 /usr/sbin/glusterd -p /var/run/glusterd.pid --log-level INFO
```

### 使用 hekiti 加入节点

#### 生产SSH-key

-C noname 为不要注释的主机名

```bash
ssh-keygen -t rsa -q -f /etc/heketi/heketi_key -N '' -C noname
```

#### 公钥copy到各主机

追加到 ~/.ssh/authorized_keys文件中，注意此文件为 600

```bash
scp heketi_key.pub 192.168.10.11X:~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

#### 创建私钥 secret

```bash
# copy heketi_key文件内容到https://tool.oschina.net/encrypt?type=3 在线base64
# 把计算的值放在 heketi_key.yml文件中
k apply -f heketi_key.yml
```

#### 创建heketi.json的secret

```bash
# copy heketi.json 文件内容到 https://tool.oschina.net/encrypt?type=3 在线base64
# 把计算的值放在 heketi.json.yml文件中
k apply -f heketi.json.yml
```

> 注意：docker版本的 glusterfs 的 sshd 开放的是端口 2222
>

## 使用heketi创建节点，数据copy，以下内容忽略

---

### 将分布式存储主机加入到信任主机池并查看加入的主机状态

随便在一个开启glusterfs服务的主机上将其他主机加入到一个信任的主机池里，这里选择113节点

```bash
[root@node01 ~]# gluster peer probe vmsrv-010-122
peer probe: success. 
[root@node01 ~]# gluster peer probe vmsrv-010-123
peer probe: success. 
[root@node01 ~]# gluster peer probe vmsrv-010-221
peer probe: success.
```

查看主机池中主机的状态

```bash
[root@vmsrv-010-113 ~]# gluster peer status
Number of Peers: 3

Hostname: vmsrv-010-122
Uuid: 0577218b-8fb1-4b6f-9f7a-d0cec2c30605
State: Peer in Cluster (Connected)

Hostname: vmsrv-010-123
Uuid: 07492389-77e7-45ee-becc-e4d7564c9b66
State: Peer in Cluster (Connected)

Hostname: vmsrv-010-221
Uuid: f2593f4f-6b27-4436-a383-fef372f76d37
State: Peer in Cluster (Connected)
```

## 配置分布式复制卷

最少需要4台服务器才能创建,[**生产场景推荐使用此种方式**]

- 将原有的复制卷gv2进行扩容，使其成为分布式复制卷；

- 要扩容前需停掉gv2

- force是强制在系统盘执行，这个在gluster的默认情况下是不允许的，生产环境下也尽可能的与系统盘分开，如果必须这样请使用force 


```bash
$ mkdir /bricks/brick2/k8s_brick
$ gluster volume create k8s-volume replica 2 vmsrv-010-113:/bricks/brick2/k8s_brick vmsrv-010-122:/bricks/brick2/k8s_brick vmsrv-010-123:/bricks/brick2/k8s_brick vmsrv-010-221:/bricks/brick2/k8s_brick

$ gluster volume start k8s-volume
$ gluster volume info k8s-volume
------------------------------------
Volume Name: k8s-volume
Type: Distributed-Replicate #这里显示是分布式复制卷，是在 gv2 复制卷的基础上增加 2 块 brick 形成的
Volume ID: 8982c330-3694-4621-9040-52ef0288bc63
Status: Started
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: vmsrv-010-113:/bricks/brick2/k8s_brick
Brick2: vmsrv-010-122:/bricks/brick2/k8s_brick
Brick3: vmsrv-010-123:/bricks/brick2/k8s_brick
Brick4: vmsrv-010-221:/bricks/brick2/k8s_brick
Options Reconfigured:
performance.write-behind-window-size: 1024MB
network.ping-timeout: 10
performance.io-thread-count: 16
performance.cache-size: 4GB
features.quota-deem-statfs: on
features.inode-quota: on
features.quota: on
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```
> 注意：当你给分布式复制卷和分布式条带卷增加 bricks 时，你增加的 bricks 数目必须是复制或条带数目的倍数，例如：你给一个分布式复制卷的 replica 为 2，你在增加 bricks 的时候数量必须为2、4、6、8等。
> 扩容后进行测试，发现文件都分布在扩容前的卷中。

## 删除测试卷

```bash
gluster volume stop k8s-volume
gluster volume delete k8s-volume
```
---

## 相关命令

```bash
#为存储池添加/移除服务器节点
$ gluster peer probe
$ gluster peer detach
$ gluster peer status
#创建/启动/停止/删除卷
$ gluster volume create [stripe | replica ] [transport [tcp | rdma | tcp,rdma]] ...
$ gluster volume start
$ gluster volume stop
$ gluster volume delete

注意，删除卷的前提是先停止卷。

#查看卷信息
$ gluster volume list
$ gluster volume info [all]
$ gluster volume status [all]
$ gluster volume status [detail| clients | mem | inode | fd]

#查看本节点的文件系统信息：
$ df -lh
#查看本节点的磁盘信息：
$ fdisk -l
#卸载挂载
$ umount /dev/sdb
```

#### 磁盘分区

```bash
[root@gluster-node-1 mingguilu]# fdisk /dev/sdb 
欢迎使用 fdisk (util-linux 2.23.2)。

更改将停留在内存中，直到您决定将更改写入磁盘。
使用写入命令前请三思。

Device does not contain a recognized partition table
使用磁盘标识符 0x896a8f0e 创建新的 DOS 磁盘标签。

命令(输入 m 获取帮助)：m
命令操作
a   toggle a bootable flag
b   edit bsd disklabel
c   toggle the dos compatibility flag
d   delete a partition
g   create a new empty GPT partition table
G   create an IRIX (SGI) partition table
l   list known partition types
m   print this menu
n   add a new partition
o   create a new empty DOS partition table
p   print the partition table
q   quit without saving changes
s   create a new empty Sun disklabel
t   change a partition's system id
u   change display/entry units
v   verify the partition table
w   write table to disk and exit
x   extra functionality (experts only)

命令(输入 m 获取帮助)：n
Partition type:
p   primary (0 primary, 0 extended, 4 free)
e   extended
Select (default p): p
分区号 (1-4，默认 1)：
起始 扇区 (2048-41943039，默认为 2048)：
将使用默认值 2048
Last 扇区, +扇区 or +size{K,M,G} (2048-41943039，默认为 41943039)：+300G
分区 1 已设置为 Linux 类型，大小设为 300 GiB

命令(输入 m 获取帮助)：w
The partition table has been altered!

Calling ioctl() to re-read partition table.
正在同步磁盘。

[root@gluster-node-2 mingguilu]# fdisk -l | grep sdb
磁盘 /dev/sdb：21.5 GB, 21474836480 字节，41943040 个扇区
/dev/sdb1            2048    41943039    20970496   83  Linux
```

## 磁盘格式化

```bash
mkfs.xfs -i size=512 /dev/sdb1
```

## 删除分区

```bash
fdisk /dev/sdb
m  
d  
1  
d 
```

### Glusterfs调优

```bash
# 开启 指定 volume 的配额
$ gluster volume quota k8s-volume enable

# 限制 指定 volume 的配额
$ gluster volume quota k8s-volume limit-usage / 60GB

# 设置 cache 大小, 默认32MB
$ gluster volume set k8s-volume performance.cache-size 4GB

# 设置 io 线程, 太大会导致进程崩溃
$ gluster volume set k8s-volume performance.io-thread-count 16

# 设置 网络检测时间, 默认42s
$ gluster volume set k8s-volume network.ping-timeout 10

# 设置 写缓冲区的大小, 默认1M
$ gluster volume set k8s-volume performance.write-behind-window-size 1024MB
```

## 客户端挂载

1.安装glusterfs

```bash
[root@NK05 ~]# yum -y install glusterfs glusterfs-fuse
```



2.挂载glusterfs

```bash
[root@NK05 ~]# mount -t glusterfs NK01:/drv1 /mnt/drv1
[root@NK05 ~]# df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G  1.3G   49G   3% /
devtmpfs                 1.9G     0  1.9G   0% /dev
tmpfs                    1.9G     0  1.9G   0% /dev/shm
tmpfs                    1.9G  8.6M  1.9G   1% /run
tmpfs                    1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda1               1014M  145M  870M  15% /boot
/dev/mapper/centos-home  142G   33M  142G   1% /home
tmpfs                    378M     0  378M   0% /run/user/0
NK01:/drv1               283G  2.9G  280G   2% /mnt/drv1
```



3.设置开机自动挂载

```bash
[root@NK05 ~]# echo 'NK01:/drv1   /mnt/drv1  glusterfs defaults,_netdev 0 0' >> /etc/fstab
```

---

## 安装 Hekeit

Heketi 是用来管理 GlusterFS 卷的生命周期的，并提供了一个 RESTful API 接口供 Kubernetes 调用，因为 GlusterFS 没有提供 API 调用的方式，所以我们借助 heketi，通过 Heketi，Kubernetes 可以动态配置 GlusterFS 卷。以下将介绍如何在 glusterfs-server1 安装 Heketi（v7.0.0）

#### 下载 Hekeit Installer：

```bash
root@glusterfs-server1:~# wget https://github.com/heketi/heketi/releases/download/v7.0.0/heketi-v7.0.0.linux.amd64.tar.gz
```

#### 解压并安装 Heketi：

```bash
root@glusterfs-server1:~# tar -zxvf heketi-v9.0.0.linux.amd64.tar.gz
root@glusterfs-server1:~# cd heketi/
root@glusterfs-server1:~/heketi# cp heketi /usr/bin/
root@glusterfs-server1:~/heketi# cp heketi-cli /usr/bin
```

### 配置 Heketi

1. 将 Heketi 纳入 `systemd`管理：

   ```bash
   root@glusterfs-server1:~# vi /lib/systemd/system/heketi.service
   [Unit]
   Description=Heketi Server
   [Service]
   Type=simple
   WorkingDirectory=/var/lib/heketi
   ExecStart=/usr/bin/heketi --config=/etc/heketi/heketi.json
   Restart=on-failure
   StandardOutput=syslog
   StandardError=syslog
   [Install]
   WantedBy=multi-user.target
   ```

2. 创建文件夹：

```bash
root@glusterfs-server1:~# mkdir -p /var/lib/heketi
root@glusterfs-server1:~# mkdir -p /etc/heketi
```
3. 创建 Heketi 配置文件：vim /etc/heketi/heketi.json
   - "loglevel" : "warning" ---调整日志输出级别
   - "db": "/var/lib/heketi/heketi.db"---定义heketi数据库文件位置
   - "executor"---修改执行插件为ssh，并配置ssh的所需证书，注意要能对集群中的机器免密ssh登陆，使用ssh-copy-id把pub key拷到每台glusterfs服务器上。需要说明的是，heketi有三种executor，分别为mock、ssh、kubernetes，建议在测试环境使用mock，**生产环境使用ssh，当glusterfs以容器的方式部署在kubernetes上时，才使用kubernetes。**我们这里将glusterfs和heketi独立部署，使用ssh的方式。
   - "use_auth"---允许认证
```json
{
  "_port_comment": "Heketi Server Port Number",
  "port": "8080",

	"_enable_tls_comment": "Enable TLS in Heketi Server",
	"enable_tls": false,

	"_cert_file_comment": "Path to a valid certificate file",
	"cert_file": "",

	"_key_file_comment": "Path to a valid private key file",
	"key_file": "",


  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": false,

  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "key": "jrtz0755"
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "key": "jrtz0755"
    }
  },

  "_backup_db_to_kube_secret": "Backup the heketi database to a Kubernetes secret when running in Kubernetes. Default is off.",
  "backup_db_to_kube_secret": false,

  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/root/.ssh/id_rsa",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab",
      "backup_lvm_metadata": false
    },

    "_kubeexec_comment": "Kubernetes configuration",
    "kubeexec": {
      "host" :"https://kubernetes.host:8443",
      "cert" : "/path/to/crt.file",
      "insecure": false,
      "user": "kubernetes username",
      "password": "password for kubernetes user",
      "namespace": "OpenShift project or Kubernetes namespace",
      "fstab": "/etc/fstab",
      "backup_lvm_metadata": false
    },

    "_db_comment": "Database file name",
    "db": "/var/lib/heketi/heketi.db",
    "brick_max_size_gb" : 1024,
    "brick_min_size_gb" : 1,
    "max_bricks_per_volume" : 33,

     "_refresh_time_monitor_gluster_nodes": "Refresh time in seconds to monitor Gluster nodes",
    "refresh_time_monitor_gluster_nodes": 120,

    "_start_time_monitor_gluster_nodes": "Start time in seconds to monitor Gluster nodes when the heketi comes up",
    "start_time_monitor_gluster_nodes": 10,

    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
    "loglevel" : "debug",

    "_auto_create_block_hosting_volume": "Creates Block Hosting volumes automatically if not found or exsisting volume exhausted",
    "auto_create_block_hosting_volume": true,

    "_block_hosting_volume_size": "New block hosting volume will be created in size mentioned, This is considered only if auto-create is enabled.",
    "block_hosting_volume_size": 500
  }
}
```

### 配置 SSH 免密登录

1. 创建密钥，提示 “Enter passphrase” 时，直接回车键，口令即为空：

   ```bash
   root@glusterfs-server1:~# ssh-keygen
   ```

2. 拷贝密钥到各个 GlusterFS 节点，按照提示输入密钥：

   ```bash
   root@glusterfs-server1:~# ssh-copy-id root@192.168.10.113
   root@glusterfs-server1:~# ssh-copy-id root@192.168.10.122
   root@glusterfs-server1:~# ssh-copy-id root@192.168.10.123
   root@glusterfs-server1:~# ssh-copy-id root@192.168.10.221
   ```

3. 验证免密登录，即 glusterfs-server1 无需输入密码可以登录 glusterfs-server1 和 glusterfs-server2：

   ```bash
   root@glusterfs-server1:~# ssh root@192.168.10.113
   root@glusterfs-server1:~# ssh root@192.168.10.122
   ```

## 配置ssh密钥

在上面我们配置heketi的时候使用了ssh的executor，那么就需要heketi服务器能通过ssh密钥的方式连接到所有glusterfs节点进行管理操作，所以需要先生成ssh密钥

```bash
ssh-keygen -t rsa -q -f /etc/heketi/heketi_key -N ''
chmod 600 /etc/heketi/heketi_key.pub

# ssh公钥传递，这里只以一个节点为例
ssh-copy-id -i /etc/heketi/heketi_key.pub root@192.168.10.113

# 验证是否能通过ssh密钥正常连接到glusterfs节点

ssh -i /etc/heketi/heketi_key root@192.168.10.113
```

## 启动Heketi(一)，暂时不用这种方式
```bash
nohup heketi -config=/etc/heketi/heketi.json &
```
## 启动Heketi(二)

在 glusterfs-server1 安装 Heketi 后，需要启动 Heketi，Heketi 的状态 "Active" 显示 `active (running) ...`则说明成功启动：

```bash
systemctl daemon-reload && systemctl enable heketi && systemctl start heketi
```
## 启动Heketi(三)
在我实际生产中，使用docker-compose来管理heketi，而不直接手动启动，下面直接给出docker-compose配置示例：
```bash
version: "2"
services:
  heketi:
    container_name: heketi
    image: dk-reg.op.douyuyuba.com/library/heketi:5
    volumes:
      - "/etc/heketi:/etc/heketi"
      - "/var/lib/heketi:/var/lib/heketi"
      - "/etc/localtime:/etc/localtime"
    network_mode: host
```
查看状态

```bash
systemctl status heketi
journalctl -u heketi.service -f -n 500
```

## 编辑拓扑文件

参考如下步骤编辑 Heketi 的拓扑文件，以下所有 IP 地址应替换为您安装环境的实际主机 IP 地址。GlusterFS 服务端将数据存储至 `/dev/sdb`块设备中，以下 `"/dev/sdb"`可按实际情况修改：

**Heketi 要求在每个 GlusterFS 节点上配备裸磁盘 device，不支持文件系統**，一般如下配置，可以通过 fdisk –l 命令查看。

#### 删除挂载

```json
#使用file -s 查看硬盘如果显示为data则为原始块设备。如果不是data类型，可先用pvcreate，pvremove来变更。
[root@node-04 ~]# file -s /dev/sdb
/dev/sdb: x86 boot sector, code offset 0xb8
[root@node-04 ~]# pvcreate /dev/sdb
WARNING: dos signature detected on /dev/sdc at offset 510. Wipe it? [y/n]: y
Wiping dos signature on /dev/sdb.
Physical volume "/dev/sdb" successfully created.
[root@node-04 ~]# pvremove /dev/sdb
Labels on physical volume "/dev/sdb" successfully wiped.
[root@node-04 ~]# file -s /dev/sdb 
/dev/sdb: data
-------------------------------------------------------------
# Can't initialize physical volume "/dev/sdb" of volume group "vg_cc29d32679317f955ebb22461ae831da" without -ff
vgremove vg_cc29d32679317f955ebb22461ae831da
```

```bash
export HEKETI_CLI_SERVER=http://localhost:8080
heketi-cli topology load --json=/etc/heketi/topology.json
#或者
heketi-cli --server http://localhost:8080 topology load --json=/etc/heketi/topology.json
```

## 验证 Heketi 安装
```bash
heketi-cli volume list --secret jrtz0755 --user admin 
```

## 实际部署中遇到的问题

1.heketi-client导入GlusterFS集群配置信息时遇到的问题

正常时的输出日志：

```
#heketi-cli topology load --json=topology-sample.json
Creating cluster ... ID: 5b930ef6081fd22e895c25a3dfb0c516
    Allowing file volumes on cluster.
    Allowing block volumes on cluster.
    Creating node 10.30.1.15 ... ID: b120572be40db6c1d979c3903876430b
        Adding device /dev/sdb ... OK
    Creating node 10.30.1.16 ... ID: 7ce13ffc5eabe64a3791e93233fd3c1a
        Adding device /dev/sdb ... OK
    Creating node 10.30.1.17 ... ID: f9abdc2e5d4cfa17c035a97f984a1a3b
        Adding device /dev/sdb ... OK
```

当磁盘文件系统已经存在时的错误日志

```
[root@10-211-105-109 heketi]# heketi-cli --server http://localhost:8088 topology load --json=/etc/heketi/topology.json
	Creating node 10.211.105.200 ... ID: 6c0476a0b495bc67c2e6b181ca2f0813
		Adding device /dev/sda5 ... Unable to add device: Can't open /dev/sda5 exclusively.  Mounted filesystem?
```

当节点是已经位于其他集群中时的报错

```
[root@10-211-105-109 heketi]# heketi-cli --server http://localhost:8088 topology load --json=/etc/heketi/topology.json
	Creating node 10.211.106.9 ... Unable to create node: peer probe: failed: 10.211.106.9 is either already part of another cluster or having volumes configured
```

2.当使用heketi创建卷时，创建失败后会怎样

heketi记录的节点空间会减少，并且无法释放。也看不到卷信息。最终的结果是节点空间显示使用了很多，但是看不到任何的使用情况。这会不会是一个BUG?