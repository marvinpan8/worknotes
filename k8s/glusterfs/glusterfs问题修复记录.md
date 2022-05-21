# BUG修复记录

> <https://blog.csdn.net/weixin_34202952/article/details/91753519>

## 前言

- 常用命令

  ```bash
  # 查卷状态
  gluster volume status vol_6b5338ed3f91c657ade4f1c49124a83b
  # 查日志
  tail -f -n 500 /var/log/glusterfs/glusterd.log
  tail -f -n 500 /var/log/glusterfs/glustershd.log
  tail -f -n 500 /var/log/glusterfs/bricks/var-lib-heketi-mounts-vg_099dabdc48c0ce612fb3a4e29719398b-brick_78c017e9a79f93fb658f0010fcd8e95d-brick.log
  # 查brick进程
  ps -ef | grep /usr/sbin/glusterfsd |wc -l
  ps aux | grep glusterfsd | grep brick-port 
  ```

- **如果挂载正常，重启glusterd，还是启动不了主机的brick进程，则重启docker正常**

- docker部署的开放了2222端口，直接即可

  ```bash
  ssh -p 2222 192.168.10.113
  ```

#### 三个挂载主机的重要目录

- **/var/lib/glusterd：** 配置文件，**很重要**

  - `vols/vol_xxxxxxxxxxxxxxx/vol_xxxxxxxxxxxxxxx.192.168.10.122.var-lib-heketi-mounts-vg_xxx-brick_xxx-brick.vol`  此有vol下的brick配置文件，有几个brick,就有几个此文件
  - 其中在上面路径下的 `brick/192.168.10.122_-var-lib-heketi-mounts-vg_xxxxxx-brick_xxxxx-brick`  此文件为brick进程**启动配置文件**
  - glustershd/glustershd-server.vol 此文件glustershd-server进程启动文件

- **/var/log/glusterfs**：日志，**很重要**

  - **glusterd.log主程序日志**
  - **glustershd.log  每个brick进程日志**
  - **bricks文件夹下为具体详细的brick启动错误日志等**

- **/etc/glusterfs：**配置文件目录，基本不变

- 目前的卷

  ```bash
  $ gluster volume list
  heketidbstorage --- heketi
  vol_1073f8239a178186946d391ad207ac44 --- Harbor
  vol_2cdd0db3a6903d9b8deb73ff5e19e751 --- drone
  vol_6b5338ed3f91c657ade4f1c49124a83b --- grafana
  vol_ad9fb74d81366e566b0fbf6f1bae9fd8 --- aibot
  ```


## 1. 内存太大恢复
```bash
# 从124机器进入 gluster实例机器
$ ssh 192.168.10.143 -p 2222
# 重新启动程序
$ systemctl restart glusterd.service 
```

## 2. brick故障恢复

### 结束故障brick的进程

**查看 volume 状态，是否 brick 进程全部都开启**

```bash
#  注意Online项全部为“Y”，每个brick 一个进程
$ gluster volume status vol_1073f8239a178186946d391ad207ac44
Status of volume: gv0
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick GH01:/data/brick1/gv0                 N/A       N/A        N       N/A
Brick GH02:/data/brick1/gv0                 49153     0          Y       1379
Brick GH03:/data/brick1/gv0                 49153     0          Y       1281
Brick GH04:/data/brick1/gv0                 49153     0          Y       1375
Self-heal Daemon on localhost               N/A       N/A        Y       1484
Self-heal Daemon on GH02                    N/A       N/A        Y       1453
Self-heal Daemon on GH03                    N/A       N/A        Y       1443
Self-heal Daemon on GH04                    N/A       N/A        Y       1444

Task Status of Volume gv0
------------------------------------------------------------------------------
There are no active volume tasks
```

> 注：如果状态Online项为“**N**”的GH01**存在PID号**（不显示N/A）应当使用如下命令结束掉进程方可继续下面步骤。`kill -15 pid`
>

### 查看主机上的挂载目录

```bash
$ df -h
Filesystem  Size  Used Avail Use%   Mounted on
....
/dev/mapper/vg_bc6a6d0b993334071460344edda2c14d-brick_54c0cb6ad54b612bcb19636147f646f5  2.0G   33M  2.0G   2% /var/lib/heketi/mounts/vg_bc6a6d0b993334071460344edda2c14d/brick_54c0cb6ad54b612bcb19636147f646f5
/dev/mapper/vg_bc6a6d0b993334071460344edda2c14d-brick_8158f70cd8bde0079efe7b34673e5eff   25G   35M   25G   1% /var/lib/heketi/mounts/vg_bc6a6d0b993334071460344edda2c14d/brick_8158f70cd8bde0079efe7b34673e5eff
/dev/mapper/vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_4ec5bfb41025cde4d0986f9188bc4e63 1014M   53M  962M   6% /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_4ec5bfb41025cde4d0986f9188bc4e63
```

- ~~**软链接： **~~

  其中 `/dev/mapper/vg_2f11cd7f4e0164ac2edb3832bf3d0906-brick_6e50b9f96537440c28df0ff63d56bf83 `为一个块设备的软链接（l 开头的文件）。

  如下，挂载到/dev/dm-37的块设备

```bash
# 如果块链接失效，一般不用软链接，而是用硬链接
$ ln -s ../dm-39 vg_fbce95ef571b0e6e34d8a888feb3758c-brick_33ad154221d44c66322f801215d42f51
-------------------------------------------------------------------
lrwxrwxrwx 1 root root  8 Feb 17 12:08 vg_099dabdc48c0ce612fb3a4e29719398b-brick_19c7e5e21e6cc01ff3c645055f244833 -> ../dm-37
```

- **硬链接(推荐)：**

  有的系统 vg_099dabdc48c0ce612fb3a4e29719398b-brick_19c7e5e21e6cc01ff3c645055f244833 直接就是块设备（b开头的文件）**注意 brick前面是 -**

```bash
ln -d ../dm-39 vg_fbce95ef571b0e6e34d8a888feb3758c-brick_33ad154221d44c66322f801215d42f51
```

从前面对比，可以发现**3个块设备**未 `mount`

```bash
$ fdisk -l
......
Disk /dev/mapper/vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_4ec5bfb41025cde4d0986f9188bc4e63: 1073 MB, 1073741824 bytes, 2097152 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
------------------------------------------------harbor umount
Disk /dev/mapper/vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_bdb2bd668c92b64f68835518ec94a398: 268.4 GB, 268435456000 bytes, 524288000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
------------------------------------------------aibot umount
Disk /dev/mapper/vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_c0505974a97c0dc5c5ef178a24e2556b: 10.7 GB, 10737418240 bytes, 20971520 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
--------------------------------------------------
Disk /dev/mapper/vg_bc6a6d0b993334071460344edda2c14d-brick_54c0cb6ad54b612bcb19636147f646f5: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes

Disk /dev/mapper/vg_bc6a6d0b993334071460344edda2c14d-brick_8158f70cd8bde0079efe7b34673e5eff: 26.8 GB, 26843545600 bytes, 52428800 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
--------------------------------------------harbor umount
Disk /dev/dm-59: 268.4 GB, 268435456000 bytes, 524288000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 262144 bytes / 262144 bytes
```

使用如下命令 查看，对应的具体详细是哪个 pvc

`h topology info` 

### 重新挂载

```bash
# 先进入对应143主机的容器
$ k exec -it glusterfs-hjjtx  bash
# 查看文件系统类型
$ findmnt
-------------------------------------
# 重要一步
$ mount /dev/mapper/vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_c0505974a97c0dc5c5ef178a24e2556b /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b 
# 重新启动程序
$ systemctl restart glusterd.service 
```

查看其他启动 brick 进程的CMD详细信息

```bash
 $ ps -ef | grep /usr/sbin/glusterfsd |wc -l
 -------------------------------------
/usr/sbin/glusterfsd -s 192.168.10.143 --volfile-id vol_ad9fb74d81366e566b0fbf6f1bae9fd8.192.168.10.143.var-lib-heketi-mounts-vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_c0505974a97c0dc5c5ef178a24e2556b-brick -p /var/run/gluster/vols/vol_ad9fb74d81366e566b0fbf6f1bae9fd8/192.168.10.143-var-lib-heketi-mounts-vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_c0505974a97c0dc5c5ef178a24e2556b-brick.pid -S /var/run/gluster/62c4581a56c19905.socket --brick-name /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick -l /var/log/glusterfs/bricks/var-lib-heketi-mounts-vg_6c97e6ce39514cab79b1940c9bd5dd8c-brick_c0505974a97c0dc5c5ef178a24e2556b-brick.log --xlator-option *-posix.glusterd-uuid=25b6ed92-6355-425a-adc9-25202419bf9a --process-name brick --brick-port 49158 --xlator-option vol_ad9fb74d81366e566b0fbf6f1bae9fd8-server.listen-port=49158
```
## 3. vg修复

在 `/dev`下有vg_xxxxxx文件夹，此文件夹下检查软链接brick地址

```bash
#查看挂载情况
$ df -l
$ ln -s /dev/mapper/vg_bc6a6d0b993334071460344edda2c14d-brick_9a1d5643dad5442cbb3ae3a0b51b34c1 brick_9a1d5643dad5442cbb3ae3a0b51b34c1
```

## 4. gluster peer hostname修复

使用`gluster peer status` 查看状态，Hostname值不是IP，需要重新定义为IP，Other names为真正的**hostname:  vmsrv-010-122**

```bash
[root@vmsrv-010-113 peers]# gluster peer status
Number of Peers: 2

# bug hostname 
Hostname: vmsrv-010-122
Uuid: 0577218b-8fb1-4b6f-9f7a-d0cec2c30605
State: Peer in Cluster (Connected)
Other names:

Hostname: 192.168.10.143
Uuid: 25b6ed92-6355-425a-adc9-25202419bf9a
State: Peer in Cluster (Connected)
Other names:
vmsrv-010-143
```

**解决办法**：在 `/var/lib/glusterd/peers`文件夹下面，修改名为uuid（此例为284c0207-a3c4-4891-8daf-5a1718ee3b60）的配置文件

```bash
uuid=284c0207-a3c4-4891-8daf-5a1718ee3b60
state=3
hostname1=192.168.10.113
hostname2=vmsrv-010-113
```



----
## 附件
### 设置文件系统扩展属性

```bash
# 进入正常的容器里
$ k exec -it glusterfs-4szf2 bash
# 获取 brick 文件夹的扩展属性
$ cd /var/lib/heketi/mounts/vg_099dabdc48c0ce612fb3a4e29719398b/brick_599f074c2d602fb9efc7878614c70efb
# 查看 brick 属性
$ getfattr -d -m. -e hex brick
-------------------------------------------------------
getfattr: Removing leading '/' from absolute path names
# file: var/lib/heketi/mounts/vg_099dabdc48c0ce612fb3a4e29719398b
#                                         /brick_599f074c2d602fb9efc7878614c70efb/brick
trusted.afr.dirty=0x000000000000000000000000
trusted.afr.vol_ad9fb74d81366e566b0fbf6f1bae9fd8-client-1=0x000000000000000000000000
trusted.gfid=0x00000000000000000000000000000001
trusted.glusterfs.dht=0x000000010000000000000000ffffffff
trusted.glusterfs.volume-id=0xc9af8aa78bf7469f8c756a3e4163a7da
--------------------------------------------------------------------
# 进入到坏的容器
$ k exec -it glusterfs-hjjtx  bash
$ cd /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b
# 设置属性，其中 -n 表示属性名称，-v 后面接属性的储存内容。 -x 表示删除该属性数据。
setfattr -n "trusted.afr.dirty" -v "0x000000000000000000000000" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick1
setfattr -n "trusted.afr.vol_ad9fb74d81366e566b0fbf6f1bae9fd8-client-2" -v "0x000000000000000000000000" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick1
setfattr -n "trusted.gfid" -v "0x00000000000000000000000000000001" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick1
setfattr -n "trusted.glusterfs.dht" -v "0x000000010000000000000000ffffffff" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick1
setfattr -n "trusted.glusterfs.volume-id" -v "0xc9af8aa78bf7469f8c756a3e4163a7da" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick1
------删除属性--------------------
setfattr -x "trusted.afr.dirty" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick
setfattr -x "trusted.afr.vol_ad9fb74d81366e566b0fbf6f1bae9fd8-client-2" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick
setfattr -x "trusted.gfid" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick
setfattr -x "trusted.glusterfs.dht" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick
setfattr -x "trusted.glusterfs.volume-id" /var/lib/heketi/mounts/vg_6c97e6ce39514cab79b1940c9bd5dd8c/brick_c0505974a97c0dc5c5ef178a24e2556b/brick
```

### 创建新的数据目录

```
mkfs.xfs -i size=512 /dev/sdb1
```

编辑fstab

```
vim /etc/fstab
```

去掉注释：

```
/dev/sdb1 /data xfs defaults 1 2
```

重新挂载文件系统：

```
mount -a
```

增加新的数据存放文件夹（不可以与之前目录一样）

```
mkdir -p /data/brick1/gv1
```