# 监控

**1.**       **Volume Profile   命令**

Volume Profile 命令提供了一个接口，可以查看一些卷的信息，比如I/O或一些文件操作等，从而可以找出瓶颈所在。

**启动Profiling**

***命令格式：***

**gluster volume profile VOLNAME start**

For example:

```
gluster volume profile vol_1073f8239a178186946d391ad207ac44 start
```

**开启此命令会影响性能，因此建议只在做 debugging 时启用。**

**2** **查看IO信息**

***命令格式：***

**gluster volume profile VOLNAME info [nfs]**

For example:

```
gluster volume profile vol_1073f8239a178186946d391ad207ac44 info
```

**3  关闭 Profiling**

**命令格式：**

**gluster volume profile VOLNAME stop**

For example:

```
gluster volume profile vol_1073f8239a178186946d391ad207ac44 stop
```

**2.  VolumeTop 命令**

Volume Top命令可以查看 **glusterFS bricks**的一些性能指标。

 

**1  查看当前打开的文件数及最大打开的文件数**

***命令格式：***

**gluster volume top VOLNAME  open [nfs | brick BRICK-NAME] [list-cnt CNT]**

For example:

```bash
gluster volume top vol_1073f8239a178186946d391ad207ac44 open brick g1:/data list-cnt *cnt*5
```

**如果不指定 brick，Volume Top命令默认会返回100条结果，可以使用 list-cnt制返回的数量。**

**2 查看读取频率最高的文件**

**命令格式：**

**gluster volume top VOLNAMEread [nfs | brick BRICK-NAME] [list-cnt CNT]**

For example:

```bash
gluster volume top test-volume read
```

**如果不指定 brick，Volume Top命令默认会返回100条结果，可以使用 list-cnt 限制返回的数量。**

**3 查看写频率最高的文件**

**命令格式：**

**gluster volume top VOLNAMEwrite [nfs | brick BRICK-NAME] [list-cnt CNT]**

For example:

```bash
gluster volume top test-volume write list-cnt 10
```

**如果不指定brick ，Volume Top命令默认会返回100条结果，可以使用 list-cnt 限制返回的数量。**

**4 查看打开频率最高的目录**

**命令格式：**

**gluster volume top VOLNAMEopendir [nfs | brick BRICK-NAME] [list-cnt CNT]**

For example:

```bash
gluster volume top test-volume opendir brick g1:/data list-cnt *cnt*5
```

**如果不指定brick ，Volume Top命令将返回该卷上所有brick的信息**

**5 查看读取频率最高的目录**

**命令格式：**

**gluster volume top VOLNAMEreaddir [nfs | brick BRICK-NAME] [list-cnt CNT]**

For example:

```bash
gluster volume top test-volume readdir
```

**6 查看 brick 的读/写性能**

**命令格式：**

**gluster volume top VOLNAMEread-perf/write-perf [bs BLK-SIZEcount COUNT] [nfs | brick BRICK-NAME] [list-cnt CNT]**

For example:

```bash
gluster volume top test-volume read-perf
gluster volume top test-volume write-perf bs 1024 count 10brick g1:/data list-cnt 5
```

**7 列出当前所有的卷**

```bash
gluster volume list
```

**8 转储状态文件**

**可以将卷的一些内部变量及状态信息转储到文件中去，文件的默认存储路径为/var/run/gluster，文件名为 BRICK-PATH.BRICK-PID.dump**

**命令格式：**

**gluster volume statedump VOLNAME[nfs] [all|mem|iobuf|callpool|priv|fd|inode|history ]**

**mem        Dumps the memory usage and memory pool detailsof the bricks.
Iobuf       Dumps iobuf details of the bricks.
priv        Dumps private information of loaded translators.
callpool    Dumpsthe pending calls of the volume.
fd             Dumps the open file descriptor tables of the volume.
inode       Dumpsthe inode tables of the volume.
history     Dumpsthe event history of the volume**

**更改转储文件的路径**

```
gluster volume set VOLNAME server.statedump-path PATH
```


**9 查看卷的状态**

**命令格式：**

**gluster volume status [all]  VOLNAME [nfs | shd | BRICKNAME]] [detail |clients | mem | inode | fd |callpool]** 

For example:

```
gluster volume status test-volume shd
gluster volume status test-volume g1:/data detail
```
**detail          Displays additional information aboutthe bricks.**
**clients         Displays the list of clients connectedto the volume.**
**mem            Displays the memory usage andmemory pool details of the bricks.**
**inode           Displays the inode tables of thevolume.**
**fd                  Displays the open filedescriptor tables of the volume.**
**callpool       Displaysthe pending calls of the volume.**
**shd               Displays the Self-heal info ofthe volume.**