##### 注意：heketi命令在heketi机器上操作

```bash
$ export HEKETI_CLI_SERVER=http://10.254.201.109:8080
```

## 增加新机器

#### 新机器配置hosts

```bash
[root@vmsrv-010-113 ~]# vim /etc/hosts
192.168.10.113 vmsrv-010-113
192.168.10.122 vmsrv-010-122
192.168.10.143 vmsrv-010-143
```

#### 增加一个node

```bash
$ heketi-cli --user admin --secret jrtz0755 node add --zone=1 --cluster=bb829126d5fd30e3786c0b5fce852dd3 --management-host-name=192.168.10.143 --storage-host-name=192.168.10.143
#查看新增 node
$ heketi-cli --user admin --secret jrtz0755 node list
```

#### 增加device 2个

```bash
# 增加/dev/sdb
heketi-cli --user admin --secret jrtz0755 device add --name=/dev/sdb --node=c0a3ff5d09e3b202fb2cf30b2cf74581
# 增加/dev/sdc
heketi-cli --user admin --secret jrtz0755 device add --name=/dev/sdc --node=c0a3ff5d09e3b202fb2cf30b2cf74581
# 查看topology的新增node的device的ID
heketi-cli --user admin --secret jrtz0755 topology info
# 根据上面的device的ID查看device状态
heketi-cli --user admin --secret jrtz0755 device info <device id>
# 若不是online,则开启
heketi-cli --user admin --secret jrtz0755 device enable <device id>
```

#### 查看 volume

```bash
heketi-cli --user admin --secret jrtz0755 volume list
heketi-cli --user admin --secret jrtz0755 volume <ID>
# 查看副本数是 3
--------------------------------------
Durability Type: replicate
Distributed+Replica: 3
```

------

## 删除机器

#### 先离线node下的device

```bash
heketi-cli --user admin --secret jrtz0755 device disable <id>
# 查看此device状态
heketi-cli --user admin --secret jrtz0755 device info <id>
```

#### 再离线node

```bash
heketi-cli --user admin --secret jrtz0755 node disable <id>
# 查看此 node 状态
heketi-cli --user admin --secret jrtz0755 node info <id>
```

#### 然后移除node下的device 

```bash
heketi-cli --user admin --secret jrtz0755 device remove <device-id>
# 若报如下错误，请先进入glusterfs节点pod,先执行 
$ modprobe dm_thin_pool
-------------------错误-----------------------
Error: Failed to remove device, error:   /usr/sbin/modprobe failed: 1
  thin: Required device-mapper target(s) not detected in your kernel.
  Run `lvcreate --help' for more information.
```

#### 再删除node下的device 

```bash
heketi-cli --user admin --secret jrtz0755 device delete <device-id>
```

#### 最后删除node

```bash
heketi-cli --user admin --secret jrtz0755 node delete <node-id>
```

#### 删除该虚拟机磁盘挂载

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

-----

## Q&A:

**1、若虚拟机节点重启，glusterfs无法重启？**

A：增加`readinessProbe`和`livenessProbe`的启动时间，目前增加到了180秒

**2、harbor无法登陆？**

A：进入pod查看挂载目录，删除5个挂载存储po重新部署挂载即可 (registry,  chartmuseum,  jobservice,  database,  redis)

**3、查看日志**

A：在主机`/var/log/glusterfs`目录下, 主要有`glusterd.log`, `glustershd.log`

**4、pod heketi未挂载目录/var/lib/heketi到卷heketidbstorage，导致创建或删除PVC失败？**

A：删除pod, 自动重启，重新挂载生效