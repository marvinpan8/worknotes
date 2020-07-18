# GlusterFS分布式存储--推荐阅读

原文链接：https://www.cnblogs.com/huangyanqi/p/8406534.html

目录

glusterfs简介

glusterfs部署

glustefs分布式存储优化

glusterfs在企业中应用场景

参考文章地址

 

一、glusterfs简介

Glusterfs是一个开源的分布式文件系统，是Scale存储的核心，能够处理千数量级的客户端。是整合了许多存储块（server）通过Infiniband RDMA或者 Tcp/Ip方式互联的一个并行的网络文件系统。

　　特征：

- 容量可以按比例的扩展，且性能却不会因此而降低。
- 廉价且使用简单，完全抽象在已有的文件系统之上。
- 扩展和容错设计的比较合理，复杂度较低
- 适应性强，部署方便，对环境依赖低，使用，调试和维护便利

二、glusterfs安装部署

一般在企业中，采用的是分布式复制卷，因为有数据备份，数据相对安全。

　网络要求全部千兆环境，gluster 服务器至少有 2 块网卡，1 块网卡绑定供 gluster 使用，剩余一块分配管理网络 IP，用于系统管理。如果有条件购买万兆交换机，服务器配置万兆网卡，存储性能会更好。网络方面如果安全性要求较高，可以多网卡绑定。

　跨地区机房配置 Gluster，在中国网络格局下不适用。

- 注意：GlusterFS将其动态生成的配置文件存储在/var/lib/glusterd中。如果在任何时候GlusterFS无法写入这些文件（例如，当后备文件系统已满），它至少会导致您的系统不稳定的行为; 或者更糟糕的是，让您的系统完全脱机。建议为/var/log等目录创建单独的分区，以确保不会发生这种情况。

1、安装glusterfs前的环境准备　

　　**1.1、服务规划：**

| 操作系统   | IP         | 主机名       | 硬盘数量（三块）       |
| ---------- | ---------- | ------------ | ---------------------- |
| centos 7.4 | 10.0.0.101 | node1        | sdb:5G  sdc:5G  sdd:5G |
| centos 7.4 | 10.0.0.102 | node2        | sdb:5G  sdc:5G  sdd:5G |
| centos 7.4 | 10.0.0.103 | node3        | sdb:5G  sdc:5G  sdd:5G |
| centos 7.4 | 10.0.0.104 | node4        | sdb:5G  sdc:5G  sdd:5G |
| centos 7.4 | 10.0.0.105 | node5-client | sda:20G                |

 

 　　**1.2、首先关闭iptables和selinux，配置hosts文件如下**（全部glusterfs主机）

```bash
注：node01~node04所有的主机hosts文件均为此内容；同时全部修改为对应的主机名，centos7修改主机名方式：#hostnamectl set-hostname 主机名 （即为临时和永久生效）
可以使用#hostnamectl status   查看系统基本信息
```

```bash
[root@node01 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
10.0.0.101  node01
10.0.0.102  node02
10.0.0.103  node03
10.0.0.104  node04
[root@node01 ~]# systemctl stop firewalld.service       #停止firewalld
[root@node01 ~]# systemctl disable firewalld.service    #禁止firewalld开机自启
[root@node01 ~]# sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config   #关闭SELinux
[root@node01 ~]# setenforce 0
[root@node01 ~]# getenforce
Permissive
[root@node01 ~]# ntpdate time.windows.com   #同步时间
```

 　　**1.3、安装gluterfs源（全部glusterfs主机）**

```bash
[root@node01 ~]#wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo   #不是必须的
[root@node01 ~]#yum search  centos-release-gluster   #查看有哪些版本的glusterfs源    
================================================ N/S matched: centos-release-gluster =================================================
centos-release-gluster310.noarch : Gluster 3.10 (Long Term Stable) packages from the CentOS Storage SIG repository
centos-release-gluster312.noarch : Gluster 3.12 (Long Term Stable) packages from the CentOS Storage SIG repository
centos-release-gluster313.noarch : Gluster 3.13 (Short Term Stable) packages from the CentOS Storage SIG repository
centos-release-gluster36.noarch : GlusterFS 3.6 packages from the CentOS Storage SIG repository
centos-release-gluster37.noarch : GlusterFS 3.7 packages from the CentOS Storage SIG repository
centos-release-gluster38.noarch : GlusterFS 3.8 packages from the CentOS Storage SIG repository
centos-release-gluster39.noarch : Gluster 3.9 (Short Term Stable) packages from the CentOS Storage SIG repository
```

 这里我们使用glusterfs的3.12版本的源

```bash
[root@node01 ~]# yum  -y  install centos-release-gluster312.noarch
```

 　　**1.4、安装glusterfs(全部glusterfs主机)**

在安装glusterfs的时候直接指定源为glusterfs源，由于 源[centos-gluster310-test]的enable为0,所以在指定源的时候用--enablerepo来让源生效

```
 cat /etc/yum.repos.d/CentOS-Gluster-3.12.repo
```

安装glusterfs

```bash
# 暂时不用这句
$ yum -y --enablerepo=centos-gluster*-test install glusterfs-server glusterfs-cli glusterfs-geo-replication
----------------------------------------------
$ yum -y --enablerepo=centos-gluster*-test install glusterfs glusterfs-server glusterfs-fuse glusterfs-rdma glusterfs-geo-replication glusterfs-devel
```

　　**1.5、查看glusterfs版本并启动glusterfs服务**(全部glusterfs主机)

```bash
[root@node01 ~]# glusterfs -V
glusterfs 3.12.5

[root@node01 ~]# systemctl enable glusterd.service && systemctl start glusterd.service
[root@node01 ~]# systemctl status glusterd.service
[root@node01 ~]# netstat -lntup
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      1/systemd           
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      866/sshd            
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      972/master          
tcp        0      0 0.0.0.0:24007           0.0.0.0:*               LISTEN      2268/glusterd       
tcp6       0      0 :::111                  :::*                    LISTEN      1/systemd           
tcp6       0      0 :::22                   :::*                    LISTEN      866/sshd            
tcp6       0      0 ::1:25                  :::*                    LISTEN      972/master          
udp        0      0 0.0.0.0:111             0.0.0.0:*                           2266/rpcbind        
udp        0      0 0.0.0.0:745             0.0.0.0:*                           2266/rpcbind        
udp        0      0 127.0.0.1:323           0.0.0.0:*                           527/chronyd         
udp6       0      0 :::111                  :::*                                2266/rpcbind        
udp6       0      0 :::745                  :::*                                2266/rpcbind        
udp6       0      0 ::1:323                 :::*                                527/chronyd  
```

　　　**1.6、格式化磁盘(全部glusterfs主机)**

　　在每台主机上创建几块硬盘，做接下来的分布式存储使用

**注意：创建的硬盘要用xfs格式来格式化硬盘，如果用ext4来格式化硬盘的话，对于大于16TB空间格式化就无法实现了。所以这里要用xfs格式化磁盘(centos7默认的文件格式就是xfs)，并且xfs的文件格式支持PB级的数据量**

　如果是centos6默认是不支持xfs的文件格式，要先安装xfs支持包

```bash
# centos6专用
$ yum install xfsprogs -y
```

　　用`fdisk -l` 查看磁盘设备，例如查看data-1-1的磁盘设备，这里的sdc、sdd、sde是新加的硬盘

```bash
[root@node01 ~]# fdisk -l
Disk /dev/sdb: 5368 MB, 5368709120 bytes
Disk /dev/sdc: 5368 MB, 5368709120 bytes
Disk /dev/sdd: 5368 MB, 5368709120 bytes
```

　　　特别说明：

```bash
#如果磁盘大于 2T 的话就用 parted 来分区，这里我们不用分区（可以不分区）；
#做分布式文件系统的时候数据盘一般不需要做 RAID，一般系统盘会做 RAID 1；
#如果有raid卡的话，最好用上，raid卡有数据缓存功能，也能提高磁盘的iops，最好的话，用RAID 5；
#如果都不做raid的话，也是没问题的，glusterfs也是可以保证数据的安全的。

#这里使用官方推荐的格盘方式：http://docs.gluster.org/en/latest/Quick-Start-Guide/Quickstart/#purpose-of-this-document
---------------------------------------------------
[root@node01 ~]# mkfs.xfs  -i size=512 /dev/sdb1
[root@node01 ~]# mkfs.xfs  -i size=512 /dev/sdb2
```

　　在四台机器上创建挂载块设备的目录，挂载硬盘到目录

```bash
mkdir -p /bricks/brick{1..3}
echo '/dev/sdb /bricks/brick1 xfs defaults 0 0' >> /etc/fstab
echo '/dev/sdc /bricks/brick2 xfs defaults 0 0' >> /etc/fstab
echo '/dev/sdd /bricks/brick3 xfs defaults 0 0' >> /etc/fstab
--------------------------------------------------------------
echo '/dev/sdb1 /bricks/brick1 xfs defaults 0 0' >> /etc/fstab
echo '/dev/sdb2 /bricks/brick2 xfs defaults 0 0' >> /etc/fstab
#将/etc/fstab的所有内容重新加载
mount -a && mount

[root@node01 ~]# df -h
/dev/sdc        5.0G   33M  5.0G   1% /data/brick2
/dev/sdd        5.0G   33M  5.0G   1% /data/brick3
/dev/sdb        5.0G   33M  5.0G   1% /data/brick1
```

注：再次说明——以上操作均在node01-node04上同时操作

 

 2、操作

 　　2.1、将分布式存储主机加入到信任主机池并查看加入的主机状态

　随便在一个开启glusterfs服务的主机上将其他主机加入到一个信任的主机池里，这里选择node01

```bash
[root@node01 ~]# gluster peer probe node02
peer probe: success. 
[root@node01 ~]# gluster peer probe node03
peer probe: success. 
[root@node01 ~]# gluster peer probe node04
peer probe: success.
```

   　　　　查看主机池中主机的状态


```bash
[root@node01 ~]# gluster peer status
Number of Peers: 3      #除本机外，还有三台主机主机池中

Hostname: node02
Uuid: 6020709d-1b46-4e2c-9cdd-c4b3bba47b4b
State: Peer in Cluster (Connected)

Hostname: node03
Uuid: 147ee557-51f1-43fe-a27f-3dae2880b5d4
State: Peer in Cluster (Connected)

Hostname: node04
Uuid: f61af299-b00d-489c-9fd9-b4f6a336a6c7
State: Peer in Cluster (Connected)
注意：一旦建立了这个池，只有受信任的成员可能会将新的服务器探测到池中。新服务器无法探测池，必须从池中探测。
```




　　2.2、创建glusterfs卷

GlusterFS 五种卷　　

- Distributed：分布式卷，文件通过 hash 算法随机分布到由 bricks 组成的卷上。
- Replicated: 复制式卷，类似 RAID 1，replica 数必须等于 volume 中 brick 所包含的存储服务器数，可用性高。
- Striped: 条带式卷，类似 RAID 0，stripe 数必须等于 volume 中 brick 所包含的存储服务器数，文件被分成数据块，以 Round Robin 的方式存储在 bricks 中，并发粒度是数据块，大文件性能好。
- Distributed Striped: 分布式的条带卷，volume中 brick 所包含的存储服务器数必须是 stripe 的倍数（>=2倍），兼顾分布式和条带式的功能。
- Distributed Replicated: 分布式的复制卷，volume 中 brick 所包含的存储服务器数必须是 replica 的倍数（>=2倍），兼顾分布式和复制式的功能。

　分布式复制卷的brick顺序决定了文件分布的位置，一般来说，先是两个brick形成一个复制关系，然后两个复制关系形成分布。

　企业一般用后两种，大部分会用分布式复制（可用容量为 总容量/复制份数），通过网络传输的话最好用万兆交换机，万兆网卡来做。这样就会优化一部分性能。它们的数据都是通过网络来传输的。

### 配置分布式卷


```
#在信任的主机池中任意一台设备上创建卷都可以，而且创建好后可在任意设备挂载后都可以查看
[root@node01 ~]# gluster volume create gv1 node01:/data/brick1 node02:/data/brick1 force      #创建分布式卷
volume create: gv1: success: please start the volume to access data
[root@node01 ~]# gluster volume start gv1    #启动卷gv1
volume start: gv1: success
[root@node01 ~]# gluster volume info gv1     #查看gv1的配置信息
 
Volume Name: gv1
Type: Distribute  #分布式卷
Volume ID: 85622964-4b48-47d5-b767-d6c6f1e684cc
Status: Started
Snapshot Count: 0
Number of Bricks: 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick1
Brick2: node02:/data/brick1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
[root@node01 ~]# mount -t glusterfs 127.0.0.1:/gv1 /opt   #挂载gv1卷
[root@node01 ~]# df -h
文件系统        容量  已用  可用 已用% 挂载点/dev/sdc        5.0G   33M  5.0G    1% /data/brick2
/dev/sdd        5.0G   33M  5.0G    1% /data/brick3
/dev/sdb        5.0G   33M  5.0G    1% /data/brick1
127.0.0.1:/gv1   10G   65M   10G    1% /opt     #连个设备容量之和
[root@node01 ~]# cd /opt/
[root@node01 opt]# touch {a..f}      #创建测试文件
[root@node01 opt]# ll
总用量 0
-rw-r--r--. 1 root root 0 2月   2 23:59 a
-rw-r--r--. 1 root root 0 2月   2 23:59 b
-rw-r--r--. 1 root root 0 2月   2 23:59 c
-rw-r--r--. 1 root root 0 2月   2 23:59 d
-rw-r--r--. 1 root root 0 2月   2 23:59 e
-rw-r--r--. 1 root root 0 2月   2 23:59 f
# 在node04也可看到新创建的文件，信任存储池中的每一台主机挂载这个卷后都可以看到
[root@node04 ~]# mount -t glusterfs 127.0.0.1:/gv1 /opt
[root@node04 ~]# ll /opt/
总用量 0
-rw-r--r--. 1 root root 0 2月   2 2018 a
-rw-r--r--. 1 root root 0 2月   2 2018 b
-rw-r--r--. 1 root root 0 2月   2 2018 c
-rw-r--r--. 1 root root 0 2月   2 2018 d
-rw-r--r--. 1 root root 0 2月   2 2018 e
-rw-r--r--. 1 root root 0 2月   2 2018 f
[root@node01 opt]# ll /data/brick1/
总用量 0
-rw-r--r--. 2 root root 0 2月   2 23:59 a
-rw-r--r--. 2 root root 0 2月   2 23:59 b
-rw-r--r--. 2 root root 0 2月   2 23:59 c
-rw-r--r--. 2 root root 0 2月   2 23:59 e
[root@node02 ~]# ll /data/brick1
总用量 0
-rw-r--r--. 2 root root 0 2月   2 23:59 d
-rw-r--r--. 2 root root 0 2月   2 23:59 f
#文件实际存在位置node01和node02上的/data/brick1目录下,通过hash分别存到node01和node02上的分布式磁盘上
```

### 配置复制卷


```
注：复制模式，既AFR, 创建volume 时带 replica x 数量: 将文件复制到 replica x 个节点中。
这条命令的意思是使用Replicated的方式，建立一个名为gv2的卷(Volume)，存储块(Brick)为2个，分别为node01:/data/brick2和node02:/data/brick2；
fore为强制创建：因为复制卷在双方主机通信有故障再恢复通信时容易发生脑裂。本次为实验环境，生产环境不建议使用。

[root@node01 ~]# gluster volume create gv2 replica 2 node01:/data/brick2 node02:/data/brick2 force  
volume create: gv2: success: please start the volume to access data
[root@node01 ~]# gluster volume start gv2    #启动gv2卷
volume start: gv2: success
[root@node01 ~]# gluster volume info gv2   #查看gv2信息
Volume Name: gv2
Type: Replicate   #复制卷
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
[root@node01 ~]# mount -t glusterfs 127.0.0.1:/gv2 /mnt
[root@node01 ~]# df -h
文件系统        容量  已用  可用 已用% 挂载点
/dev/sdc        5.0G   33M  5.0G    1% /data/brick2
/dev/sdd        5.0G   33M  5.0G    1% /data/brick3
/dev/sdb        5.0G   33M  5.0G    1% /data/brick1
127.0.0.1:/gv1   10G   65M   10G    1% /opt
127.0.0.1:/gv2  5.0G   33M  5.0G    1% /mnt    #容量是总容量的一半
[root@node01 ~]# cd /mnt/
[root@node01 mnt]# touch {1..6}
[root@node01 mnt]# ll /data/brick2
总用量 0
-rw-r--r--. 2 root root 0 2月   3 01:06 1
-rw-r--r--. 2 root root 0 2月   3 01:06 2
-rw-r--r--. 2 root root 0 2月   3 01:06 3
-rw-r--r--. 2 root root 0 2月   3 01:06 4
-rw-r--r--. 2 root root 0 2月   3 01:06 5
-rw-r--r--. 2 root root 0 2月   3 01:06 6
[root@node02 ~]# ll /data/brick2
总用量 0
-rw-r--r--. 2 root root 0 2月   3 01:06 1
-rw-r--r--. 2 root root 0 2月   3 01:06 2
-rw-r--r--. 2 root root 0 2月   3 01:06 3
-rw-r--r--. 2 root root 0 2月   3 01:06 4
-rw-r--r--. 2 root root 0 2月   3 01:06 5
-rw-r--r--. 2 root root 0 2月   3 01:06 6
#创建文件的实际存在位置为node01和node02上的/data/brick2目录下，因为是复制卷，这两个目录下的内容是完全一致的。
```

### 配置条带卷

```bash
[root@node01 ~]# gluster volume create gv3 stripe 2 node01:/data/brick3 node02:/data/brick3 force
volume create: gv3: success: please start the volume to access data
[root@node01 ~]# gluster volume start gv3
volume start: gv3: success
[root@node01 ~]# gluster volume info gv3
 
Volume Name: gv3
Type: Stripe
Volume ID: 54c16832-6bdf-42e2-81a9-6b8d7b547c1a
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick3
Brick2: node02:/data/brick3
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
[root@node01 ~]# mkdir /data01
[root@node01 ~]# mount -t glusterfs 127.0.0.1:/gv3 /data01
[root@node01 ~]# df -h
文件系统        容量  已用  可用 已用% 挂载点
127.0.0.1:/gv3   10G   65M   10G    1% /data01
[root@node01 ~]# dd if=/dev/zero bs=1024 count=10000 of=/data01/10M.file
[root@node01 ~]# dd if=/dev/zero bs=1024 count=20000 of=/data01/20M.file
[root@node01 ~]# ll /data01/ -h
总用量 30M
-rw-r--r--. 1 root root 9.8M 2月   3 02:03 10M.file
-rw-r--r--. 1 root root  20M 2月   3 02:04 20M.file
*************************************************************************************
#文件的实际存放位置：
[root@node01 ~]# ll -h /data/brick3
总用量 15M
-rw-r--r--. 2 root root 4.9M 2月   3 02:03 10M.file
-rw-r--r--. 2 root root 9.8M 2月   3 02:03 20M.file
[root@node02 ~]# ll -h /data/brick3
总用量 15M
-rw-r--r--. 2 root root 4.9M 2月   3 02:03 10M.file
-rw-r--r--. 2 root root 9.8M 2月   3 02:04 20M.file
# 上面可以看到 10M 20M 的文件分别分成了 2 块（这是条带的特点），写入的时候是循环地一点一点在node01和node02的磁盘上.

#上面配置的条带卷在生产环境是很少使用的，因为它会将文件破坏，比如一个图片，它会将图片一份一份地分别存到条带卷中的brick上。
```

## 配置分布式复制卷

最少需要4台服务器才能创建,[**生产场景推荐使用此种方式**]

- 将原有的复制卷gv2进行扩容，使其成为分布式复制卷；

- 要扩容前需停掉gv2

- force是强制在系统盘执行，这个在gluster的默认情况下是不允许的，生产环境下也尽可能的与系统盘分开，如果必须这样请使用force 

  [root@node01 ~]# gluster volume stop gv2

```bash
mkdir /bricks/brick2/k8s_brick
gluster volume create k8s-volume replica 2 vmsrv-010-113:/bricks/brick2/k8s_brick vmsrv-010-122:/bricks/brick2/k8s_brick vmsrv-010-123:/bricks/brick2/k8s_brick vmsrv-010-221:/bricks/brick2/k8s_brick

gluster volume start k8s-volume
gluster volume info k8s-volume

-----------------------------------------------------------------------------
[root@node01 ~]# gluster volume add-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 force  #添加brick到gv2中
volume add-brick: success
[root@node01 ~]# gluster volume start gv2
volume start: gv2: success
[root@node01 ~]# gluster volume info gv2
 
Volume Name: gv2
Type: Distributed-Replicate #这里显示是分布式复制卷，是在 gv2 复制卷的基础上增加 2 块 brick 形成的
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Started
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Brick3: node03:/data/brick1
Brick4: node04:/data/brick1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off

注意：当你给分布式复制卷和分布式条带卷增加 bricks 时，你增加的 bricks 数目必须是复制或条带数目的倍数，
例如：你给一个分布式复制卷的 replica 为 2，你在增加 bricks 的时候数量必须为2、4、6、8等。
扩容后进行测试，发现文件都分布在扩容前的卷中。
```

### 配置分布式条带卷

```bash
#将原有的复制卷gv3进行扩容，使其成为分布式条带卷
#要扩容前需停掉gv3
[root@node01 ~]# gluster volume stop gv3
[root@node01 ~]# gluster volume add-brick gv3 stripe 2 node03:/data/brick2 node04:/data/brick2 force  #添加brick到gv3中
[root@node01 ~]# gluster volume start gv3
volume start: gv3: success
[root@node01 ~]# gluster volume info gv3
 
Volume Name: gv3
Type: Distributed-Stripe   # 这里显示是分布式条带卷，是在 gv3 条带卷的基础上增加 2 块 brick 形成的
Volume ID: 54c16832-6bdf-42e2-81a9-6b8d7b547c1a
Status: Started
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick3
Brick2: node02:/data/brick3
Brick3: node03:/data/brick2
Brick4: node04:/data/brick2
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
```

 再次注明文章出处：http://blog.csdn.net/goser329/article/details/78569257

## 分布式复制卷的最佳实践：

1. 搭建条件
- 块服务器的数量必须是复制的倍数
- 将按块服务器的排列顺序指定相邻的块服务器成为彼此的复制
    例如，8台服务器：
- 当复制副本为2时，按照服务器列表的顺序，服务器1和2作为一个复制,3和4作为一个复制,5和6作为一个复制,7和8作为一个复制
- 当复制副本为4时，按照服务器列表的顺序，服务器1/2/3/4作为一个复制,5/6/7/8作为一个复制
2. 创建分布式复制卷
    **磁盘存储的平衡**
    平衡布局是很有必要的，因为布局结构是静态的，当新的 bricks 加入现有卷，新创建的文件会分布到旧的 bricks 中，所以需要平衡布局结构，使新加入的 bricks 生效。布局平衡只是使新布局生效，并不会在新的布局中移动老的数据，如果你想在新布局生效后，重新平衡卷中的数据，还需要对卷中的数据进行平衡。

```bash
#在gv2的分布式复制卷的挂载目录中创建测试文件如下
[root@node01 ~]# df -h
文件系统        容量  已用  可用 已用% 挂载点
127.0.0.1:/gv2   10G   65M   10G    1% /mnt
[root@node01 ~]# cd /mnt/
[root@node01 mnt]# touch {x..z}
#新创建的文件只在老的brick中有，在新加入的brick中是没有的
[root@node01 mnt]# ls /data/brick2
1  2  3  4  5  6  x  y  z
[root@node02 ~]# ls /data/brick2
1  2  3  4  5  6  x  y  z
[root@node03 ~]# ll -h /data/brick1
总用量 0
[root@node04 ~]# ll -h /data/brick1
总用量 0
# 从上面可以看到，新创建的文件还是在之前的 bricks 中，并没有分布中新加的 bricks 中

# 下面进行磁盘存储平衡
[root@node01 ~]# gluster volume rebalance gv2 start
[root@node01 ~]# gluster volume rebalance gv2 status  #查看平衡存储状态
# 查看磁盘存储平衡后文件在 bricks 中的分布情况
[root@node01 ~]# ls /data/brick2
1  5  y
[root@node02 ~]# ls /data/brick2
1  5  y
[root@node03 ~]# ls /data/brick1
2  3  4  6  x  z
[root@node04 ~]# ls /data/brick1
2  3  4  6  x  z
#从上面可以看出部分文件已经平衡到新加入的brick中了
每做一次扩容后都需要做一次磁盘平衡。 磁盘平衡是在万不得已的情况下再做的，一般再创建一个卷就可以了。
```

### 移除 brick

你可能想在线缩小卷的大小，例如：当硬件损坏或网络故障的时候，你可能想在卷中移除相关的 bricks。
　　注意：当你移除 bricks 的时候，你在 gluster 的挂载点将不能继续访问数据，只有配置文件中的信息移除后你才能继续访问 bricks 中的数据。当移除分布式复制卷或者分布式条带卷的时候，移除的 bricks 数目必须是 replica 或者 stripe 的倍数。

　　但是移除brick在生产环境中基本上不做的，如果是硬盘坏掉的话，直接换个好的硬盘即可，然后再对新的硬盘设置卷标识就可以使用了，后面会演示硬件故障或系统故障的解决办法。

```bash
[root@node01 ~]# gluster volume stop gv2
[root@node01 ~]# gluster volume remove-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 force   #强制移除brick块
[root@node01 ~]# gluster volume info gv2    
Volume Name: gv2
Type: Replicate
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Stopped
Snapshot Count: 0
Number of Bricks: 1 x 2 = 2
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
# 如果误操作删除了后，其实文件还在 /storage/brick1 里面的，加回来就可以了
[root@node01 ~]# gluster volume add-brick gv2 replica 2 node03:/data/brick1 node04:/data/brick1 force  
volume add-brick: success
[root@node01 ~]# gluster volume info gv2
 
Volume Name: gv2
Type: Distributed-Replicate
Volume ID: 9f33bd9a-7096-4749-8d91-1e6de3b50053
Status: Stopped
Snapshot Count: 0
Number of Bricks: 2 x 2 = 4
Transport-type: tcp
Bricks:
Brick1: node01:/data/brick2
Brick2: node02:/data/brick2
Brick3: node03:/data/brick1
Brick4: node04:/data/brick1
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

```bash
删除卷
　　一般会用在命名不规范的时候才会删除 
[root@node01 ~]# gluster volume stop gv1
[root@node01 ~]# gluster volume delete gv
模拟误删除卷信息故障及解决办法
```

```bash
[root@node01 ~]# ls /var/lib/glusterd/vols/
gv2  gv3
[root@node01 ~]# rm -rf /var/lib/glusterd/vols/gv3 #删除卷gv3的卷信息
[root@node01 ~]# ls /var/lib/glusterd/vols/     #再查看卷信息情况如下：gv3卷信息被删除了
gv2
[root@node01 ~]# gluster volume sync node02      #因为其他节点服务器上的卷信息是完整的，比如从node02上同步所有卷信息如下：
Sync volume may make data inaccessible while the sync is in progress. Do you want to continue? (y/n) y
volume sync: success
[root@node01 ~]# ls /var/lib/glusterd/vols/    #验证卷信息是否同步过来
gv2  gv3
```

 

模拟复制卷数据不一致故障及解决办法

```bash
[root@node01 ~]# ls /data/brick2    #复制卷的存储位置的数据
1  5  y
[root@node01 ~]# rm -f /data/brick2/y 
[root@node01 ~]# ls /data/brick2
1  5
[root@node02 ~]# ls /data/brick2
1  5  y
[root@node01 ~]# gluster start gv2   #因为之前关闭了，如果未关闭可以忽略此步。
[root@node01 ~]# cat /mnt/y     #通过访问这个复制卷的挂载点的数据来同步数据
[root@node01 ~]# ls /data/brick2/     #这时候再看复制卷的数据是否同步成功
1  5  y
```

3、glustefs分布式存储优化

```
Auth_allow　　#IP访问授权；缺省值（*.allow all）；合法值：Ip地址
Cluster.min-free-disk　　#剩余磁盘空间阀值；缺省值（10%）；合法值：百分比
Cluster.stripe-block-size　　#条带大小；缺省值（128KB）；合法值：字节
Network.frame-timeout　　#请求等待时间；缺省值（1800s）；合法值：1-1800
Network.ping-timeout　　#客户端等待时间；缺省值（42s）；合法值：0-42
Nfs.disabled　　#关闭NFS服务；缺省值（Off）；合法值：Off|on
Performance.io-thread-count　　#IO线程数；缺省值（16）；合法值：0-65
Performance.cache-refresh-timeout　　#缓存校验时间；缺省值（1s）；合法值：0-61
Performance.cache-size　　#读缓存大小；缺省值（32MB）；合法值：字节

Performance.quick-read: #优化读取小文件的性能
Performance.read-ahead: #用预读的方式提高读取的性能，有利于应用频繁持续性的访问文件，当应用完成当前数据块读取的时候，下一个数据块就已经准备好了。
Performance.write-behind:先写入缓存内，在写入硬盘，以提高写入的性能。
Performance.io-cache:缓存已经被读过的、
```

3.1、优化参数调整方式

```bash
命令格式：
gluster.volume set <卷><参数>

例如：
#打开预读方式访问存储
[root@node01 ~]# gluster volume set gv2 performance.read-ahead on

#调整读取缓存的大小
[root@mystorage gv2]# gluster volume set gv2 performance.cache-size 256M
```

**3.2、监控及日常维护**

```bash
使用zabbix自带的模板即可，CPU、内存、磁盘空间、主机运行时间、系统load。日常情况要查看服务器监控值，遇到报警要及时处理。
#看下节点有没有在线
gluster volume status nfsp

#启动完全修复
gluster volume heal gv2 full

#查看需要修复的文件
gluster volume heal gv2 info

#查看修复成功的文件
gluster volume heal gv2 info healed

#查看修复失败的文件
gluster volume heal gv2 heal-failed

#查看主机的状态
gluster peer status

#查看脑裂的文件
gluster volume heal gv2 info split-brain

#激活quota功能
gluster volume quota gv2 enable

#关闭quota功能
gulster volume quota gv2 disable

#目录限制（卷中文件夹的大小）
gluster volume quota limit-usage /data/30MB --/gv2/data

#quota信息列表
gluster volume quota gv2 list

#限制目录的quota信息
gluster volume quota gv2 list /data

#设置信息的超时时间
gluster volume set gv2 features.quota-timeout 5

#删除某个目录的quota设置
gluster volume quota gv2 remove /data

备注：quota功能，主要是对挂载点下的某个目录进行空间限额。如：/mnt/gulster/data目录，而不是对组成卷组的空间进行限制。
```



**3.3、Gluster日常维护及故障处理**

注：给自己的提示：此处如有不详之处查看qq微云--linux-glusterfs文件夹

　1、硬盘故障

```bash
如果底层做了raid配置，有硬件故障，直接更换硬盘，会自动同步数据。
如果没有做raid处理方法：
参看其他博客：http://blog.51cto.com/cmdschool/1908647
```
　2、一台主机故障

[参考链接](https://www.cnblogs.com/xiexiaohua007/p/6602315.html)

一台节点故障的情况包含以下情况：

物理故障
同时有多块硬盘故障，造成数据丢失
系统损坏不可修复
解决方法：

找一台完全一样的机器，至少要保证硬盘数量和大小一致，安装系统，配置和故障机同样的 IP，安装 gluster 软件，
保证配置一样，在其他健康节点上执行命令 gluster peer status，查看故障服务器的 uuid

```bash
[root@mystorage2 ~]# gluster peer status
Number of Peers: 3

Hostname: mystorage3
Uuid: 36e4c45c-466f-47b0-b829-dcd4a69ca2e7
State: Peer in Cluster (Connected)

Hostname: mystorage4
Uuid: c607f6c2-bdcb-4768-bc82-4bc2243b1b7a
State: Peer in Cluster (Connected)

Hostname: mystorage1
Uuid: 6e6a84af-ac7a-44eb-85c9-50f1f46acef1
State: Peer in Cluster (Disconnected)
复制代码
修改新加机器的 /var/lib/glusterd/glusterd.info 和 故障机器一样

[root@mystorage1 ~]# cat /var/lib/glusterd/glusterd.info
UUID=6e6a84af-ac7a-44eb-85c9-50f1f46acef1
operating-version=30712
在信任存储池中任意节点执行

# gluster volume heal gv2 full
就会自动开始同步，但在同步的时候会影响整个系统的性能。

可以查看状态

# gluster volume heal gv2 info
```

### GlusterFS在企业中应用场景

　　理论和实践分析，GlusterFS目前主要使用大文件存储场景，对于小文件尤其是海量小文件，存储效率和访问性能都表现不佳，海量小文件LOSF问题是工业界和学术界的人工难题，GlusterFS作为通用的分布式文件系统，并没有对小文件额外的优化措施，性能不好也是可以理解的。

```bash
  Media
　　-文档、图片、音频、视频
　*Shared storage　　
　　-云存储、虚拟化存储、HPC（高性能计算）
　*Big data
　　-日志文件、RFID（射频识别）数据
```

参考文章地址

官方手册：<http://docs.gluster.org/en/latest/Quick-Start-Guide/Quickstart/#purpose-of-this-document>

官方原理说明：<http://docs.gluster.org/en/latest/Administrator%20Guide/Setting%20Up%20Volumes/>

主要查看博文：<https://www.liuliya.com/archive/733.html>

刘爱贵博士分享：<http://blog.csdn.net/liuaigui/article/details/17331557>

其他博主：<http://blog.csdn.net/goser329/article/details/78569257>

其他博主原理说明：<https://www.cnblogs.com/jicki/p/5801712.html>
