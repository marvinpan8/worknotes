# [GlusterFs卷的简单操作](https://www.cnblogs.com/zhangb8042/p/7801205.html)

### 一、创建卷

```bash
gluster volume create 

例子：
gluster volume create gv0 replica 2 server1:/data/brick1/gv0 server2:/data/brick1/gv0
```

发现报错了，这是因为我们创建的brick在系统盘，这个在gluster的默认情况下是不允许的，生产环境下也尽可能的与系统盘分开，如果必须这样请使用force 

```bash
gluster volume create gv0 replica 2 server1:/data/brick1/gv0 server2:/data/brick1/gv0 force  
```

### 二、启动卷

```bash
gluster volume start

例子：gluster volume start  gv0
```

### 三、停止卷

```bash
gluster volume stop

例子:  gluster volume stop gv0
```

### 四、 删除卷

```bash
gluster volume delete

例子：gluster volume delete  gv0
```

### 五、添加节点

```bash
gluster peer probe

例子：gluster peer probe server3
```



### 六、删除节点

```bash
gluster peer detach (移除节点，需要提前将该节点上的 brick移除)

例子：`gluster peer detach server3
```

### 七、查看卷

```bash
gluster volume list              /*列出集群中的所有卷*/

gluster volume info [all]      /*查看集群中的卷信息*/
gluster volume status [all]  /*查看集群中的卷状态*/
```



### 八、 更改卷类型

1.需要先卸载挂载的目录

`umount ／mnt`

2.停止卷

3.更改卷的类型

语法：`gluster volume set test-volume config.transport tcp,rdma OR tcp OR rdma`

### 九、重新均衡卷

```bash
语法：gluster volume rebalance <VOLNAME> fix-layout start

例子：gluster volume rebalance test-volume fix-layout start
```

### 十、收缩卷

```bash
1.开始收缩

gluster volume  remove-brick  gv0 server3:/data/brick1/gv0 server4:/data/brick1/gv0 start

2.查看迁移状态

gluster volume  remove-brick  gv0 server3:/data/brick1/gv0 server4:/data/brick1/gv0 status

3.迁移完成提交

gluster volume  remove-brick  gv0 server3:/data/brick1/gv0 server4:/data/brick1/gv0 commit
```

### 十一.GlusterFS的配额

GlusterFS目录限额，允许你根据目录或卷配置限制磁盘空间的使用量

1.开启限额

```
gluster volume  quota VolumeName enable
```

2.关闭限额

```
gluster volume  quota VolumeName disable
```

3.设置或替换磁盘限制

 3.1.根据卷限制

 

```bash
gluster volume quota VolumeName limit-usage / size
```

例子：`gluster volume quota gv0 limit-usage / 10GB`

3.2.根据目录限制

```bash
gluster volume ``quota` `VolumeName limit-usage DirectoryPath LimitSize
```