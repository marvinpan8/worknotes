# 使用heketi运维glusterfs

**注意：heketi命令在heketi机器上操作**

```bash
$ export HEKETI_CLI_SERVER=http://10.254.201.109:8080
$ export HEKETI_CLI_SERVER=http://localhost:8080
```

## 增加新机器

### 新机器配置hosts

在每台 glusterfs 主机上增加

```bash
[root@vmsrv-010-113 ~]# vim /etc/hosts
192.168.10.113 vmsrv-010-113
192.168.10.122 vmsrv-010-122
192.168.10.143 vmsrv-010-143
```
### 查看虚机磁盘挂载情况

若挂载则先删除该挂载盘，恢复为data则为原始块设备

```bash
# 使用file -s 查看硬盘如果显示为data则为原始块设备。如果不是data类型，可先用pvcreate，pvremove来变更。
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
#-------------------------------------------------------------
# Can't initialize physical volume "/dev/sdb" of volume group "vg_cc29d32679317f955ebb22461ae831da" without -ff
vgremove vg_cc29d32679317f955ebb22461ae831da
```

### 增加一个node

**使用别名**

```bash
vim .bashrc 
alias h='heketi-cli --user admin --secret jrtz0755'
source .bash_profile
```

**查找cluster, node list**

```bash
$ h cluster list
$ h node list
# 查看拓扑图，并拷贝出来查看
$ h topology info
```
afasdf

```bash
$ h node add --zone=1 --cluster=bb829126d5fd30e3786c0b5fce852dd3 --management-host-name=192.168.10.231 --storage-host-name=192.168.10.231
#查看新增 node
$ h node list
```

### 增加device 2个

```bash
# 增加/dev/sdb
h device add --name=/dev/sdb --node=b7a77726c01f5df32b018b051892fbb0
# 增加/dev/sdc
h device add --name=/dev/sdc --node=b7a77726c01f5df32b018b051892fbb0
# 查看topology的新增node的device的ID
h topology info
# 根据上面的device的ID查看device状态
h device info <device id>
# 若不是online,则开启
h device enable <device id>
```

### 创建 volume

```bash
heketi-cli volume create --cluster=bb829126d5fd30e3786c0b5fce852dd3 --disperse-data=<DISPERSION-VALUE> --durability=replicate --name=vol_1073f8239a178186946d391ad207ac44 --redundancy=<REDUNDENCY-VALUE> --replica=3 --size=500G --snapshot-factor=1.00


Create a GlusterFS volume
```

### 查看 volume

```bash
h volume list
h volume info <ID>
# 查看副本数是 3
--------------------------------------
Durability Type: replicate
Distributed+Replica: 3
```

------

## 删除机器

### 先离线node下的所有device

```bash
h device disable <id>
# 查看此device状态
h device info <id>
```

### 再离线node

```bash
h node disable <id>
# 查看此 node 状态
h node info <id>
```

### 然后移除node下的device 

```bash
h device remove <device-id>
# 若报如下错误，请先进入glusterfs节点pod,先执行 
$ modprobe dm_thin_pool
-------------------错误-----------------------
Error: Failed to remove device, error:   /usr/sbin/modprobe failed: 1
  thin: Required device-mapper target(s) not detected in your kernel.
  Run `lvcreate --help' for more information.
```

### 再删除node下的device 

```bash
h device delete <device-id>
```

### 最后删除node

```bash
h node delete <node-id>
```
-----

## 日志

### 修改日志级别

在 `/usr/lib/systemd/system/glusterd.service`文件中修改, Docker或k8s环境变量失效

配置文件在 `/etc/sysconfig/glusterd`

### 查看日志

在主机`/var/log/glusterfs`目录下, 主要有`glusterd.log`, `glustershd.log`

### 查看进程

每个brick一个进程

```bash
ps aux | grep glusterfsd | grep brick-port 
```



## Q&A:

**1、若虚拟机节点重启，glusterfs无法重启？**

A：增加`readinessProbe`和`livenessProbe`的启动时间，目前增加到了300秒

**2、harbor无法登陆？**

A：进入pod查看挂载目录，删除5个挂载存储po重新部署挂载即可 (registry,  chartmuseum,  jobservice,  database,  redis)

**3、pod heketi未挂载目录/var/lib/heketi到卷heketidbstorage，导致创建或删除PVC失败？**

A：删除pod, 自动重启，重新挂载生效

**4、查看 volume 状态，是否 brick 进程全部都开启**

A:   **结束故障brick的进程**

```bash
#  注意Online项全部为“Y”，每个brick 一个进程
$ gluster volume status，
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

注：如果状态Online项为“N”的GH01存在PID号（不显示N/A）应当使用如下命令结束掉进程方可继续下面步骤。

```bash
kill -15 pid
```

