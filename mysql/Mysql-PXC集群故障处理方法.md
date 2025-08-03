# Mysql-PXC集群故障处理方法

2021-07-09 https://blog.csdn.net/sinat_32582203/article/details/118611017

- 官网地址：https://docs.percona.com/percona-xtradb-cluster/8.0/crash-recovery.html

与主从复制不同，PXC集群更像一个逻辑整体。以下以三节点（A、B、C）为例，说明PXC集群的维护与故障恢复方法。

## 1、节点A正常关闭
​	此时B、C会收到A退出集群的消息，B、C的集群属性（例如 **wsrep_cluster_size**）、节点属性（例如**wsrep_local_index**）会自动变更，集群可正常提供服务。节点A重新启动后，会自动加入集群，集群属性随之变更。

## 2、节点A、B正常关闭
​	此时集群规模减小到1，集群仍然可以正常提供服务。但是，当节点A/B重新启动后，节点C的状态将变为Donor/Desynced，因为C必须至少向首个加入集群的节点提供状态传输。

​	如果您重启节点 A，然后重启节点 B，则可能需要确保节点 B 不使用节点 A 作为状态传输的源节点：节点 A 的 gcache 中可能没有所有需要的写入集。请在配置文件中将节点 C 指定为源节点，并启动 mysql 服务。

```properties
mysql> show status like 'wsrep_local_state_comment';
# ---------------------------------------
+---------------------------+--------+
| Variable_name             | Value  |
+---------------------------+--------+
| wsrep_local_state_comment | Synced |
+---------------------------+--------+
1 row in set (0.01 sec)
```

​	此过程中，集群可能仍旧能够读写，但速度会慢很多，这取决于状态传输期间需要发送的数据量。此外，某些负载均衡器可能会将 Donor 节点视为不可操作，因此，最好避免只有一个节点正常的情况出现。

## 3、节点A、B、C依次正常关闭
​	此时集群完全关闭，节点关闭时会将其最后执行操作的序列号写入 **/data/mysql/grastate.dat**，通过比较**seqno** 的值，可以看出节点的关闭顺序（前提是节点关闭后集群仍有数据写入，否则各节点该值相同）。**seqno值最大的节点，为最后一个关闭的节点，必须使用此节点启动集群。**启动命令为：

```properties
systemctl start mysql@bootstrap.service
```

​	此外，最后一个正常关闭的节点grastate.dat文件中 **safe_to_bootstrap** 值会被置为1，也可通过该值来判断需要使用哪个节点来启动集群。

​	示例，在对集群持续写入数据的情况下，依次关闭A、B、C，此时各节点的grastate.dat文件如下：

节点A：

```properties
# GALERA saved state
version: 2.1
uuid: 8acc13d0-def3-11eb-ae7a-c7af3f0ad825
seqno: 1360
safe_to_bootstrap: 0
```

节点B：

```properties
# GALERA saved state
version: 2.1
uuid: 8acc13d0-def3-11eb-ae7a-c7af3f0ad825
seqno: 1361
safe_to_bootstrap: 0
```

节点C：

```properties
# GALERA saved state
version: 2.1
uuid: 8acc13d0-def3-11eb-ae7a-c7af3f0ad825
seqno: 1362
safe_to_bootstrap: 1
```

​	建议在完全关闭集群前停止对集群的写入，以便所有节点的seqno能够停在同一位置。否则低位节点重新启动时，必须完成完整的SST（State Snapshot Transfer）才能加入集群，集群启动速度变慢。

## 4、节点A异常关闭
​	当集群中的某个节点因为断电、硬件故障、内核奔溃、进程奔溃、kill -9 mysql_pid等原因而异常关闭时，此时节点会无法将最后执行位置写入grastate.dat，seqno值为运行时值 -1，如下：

```properties
# GALERA saved state
version: 2.1
uuid: 8acc13d0-def3-11eb-ae7a-c7af3f0ad825
seqno: -1
safe_to_bootstrap: 0
```

​	剩余两个节点在发现A连接关闭时，将尝试重新连接它。如多次连接超时，节点A将被从集群中剔除。**节点A重新启动后，可自动加入集群，服务不受影响。**

## 5、节点A、B异常关闭
​	由于A、B异常关闭，剩余的节点C无法单独形成法定人数，集群将变为 **non primary** 状态。此时节点C上的mysql进程仍在运行并可以连接，但所有涉及数据的SQL查询都将报错：

```properties
mysql> SELECT * FROM test.sbtest1;
ERROR 1047 (08S01): WSREP has not yet prepared node for application use
```

​	此时通过systemctl start mysql将无法正常启动A、B，**必须手动指定节点C为主节点，**才可拉起A、B。方法如下（**节点C执行**）：

```properties
mysql> SET GLOBAL wsrep_provider_options='pc.bootstrap=true';
```

> **注意：此方法仅在其余节点异常关闭时才可使用，否则将会产生两个不同的集群。**

## 6、节点A、B、C均异常关闭
​	如遇数据中心断电、mysql/Galera发生致命错误导致所有节点均异常关闭，集群的数据一致性可能受到损坏，各个节点的 grastate.dat 都未能更新，显示为：

```properties
# GALERA saved state
version: 2.1
uuid: 8acc13d0-def3-11eb-ae7a-c7af3f0ad825
seqno: -1
safe_to_bootstrap: 0
```

此时无法通过该文件来判断哪一个节点是最后关闭的，也就无法判断以哪一个节点为主节点启动集群。

### 方法一
​	在这种情况下，首先需要通过如下命令检查各个节点的最后一次数据提交操作的序列号，结果示例如下：

```properties
# 节点A：
[root@test1 ~]# mysqld_safe --wsrep-recover
...
... mysqld_safe WSREP: Recovered position 8acc13d0-def3-11eb-ae7a-c7af3f0ad825:1364
...
# 节点B：
[root@test2 ~]# mysqld_safe --wsrep-recover
...
... mysqld_safe WSREP: Recovered position 8acc13d0-def3-11eb-ae7a-c7af3f0ad825:1365
...
# 节点C：
[root@test3 ~]# mysqld_safe --wsrep-recover
...
... mysqld_safe WSREP: Recovered position 8acc13d0-def3-11eb-ae7a-c7af3f0ad825:1365
...
```

​	可以看到，节点B、C的序列号值相等且大于A，代表这两个节点是最后关闭的，因为我们可以指定B、C中任意一个为主节点启动集群，修改其 **grastate.dat** 文件中 **safe_to_bootstrap** 值为1，随后按正常步骤启动集群即可（因为PXC集群的节点默认开机自启，服务器异常重启时mysql进程可能自启并卡死，因此**启动前需要先kill掉卡死的mysql进程**）。

```properties
# 节点B：
systemctl start mysql@bootstrap
# 节点C：
systemctl start mysql
# 节点A：
systemctl start mysql
```



### 方法二
如果使用PXC 5.6.19以上版本，还可以通过/data/mysql/gvwstate.dat文件来判断节点关闭前的集群状态。

新版本及以后新增了一个pc.recovery选项（https://www.percona.com/doc/percona-xtradb-cluster/5.7/wsrep-provider-index.html#pc.recovery），该选项默认启用，用于保存primary集群状态到每个节点的gvwstate.dat文件中。在上述示例中，各个节点的gvwstate.dat文件内容如下：

```properties
# 节点A：
[root@test1 mysql]# more gvwstate.dat
my_uuid: 052b3a03-e072-11eb-b713-e3fd5bcb270c
#vwbeg
view_id: 3 052b3a03-e072-11eb-b713-e3fd5bcb270c 15
bootstrap: 0
member: 052b3a03-e072-11eb-b713-e3fd5bcb270c 0
member: 6c9f69d0-e072-11eb-80bf-ce66d30836b1 0
member: 969e273e-e072-11eb-98eb-eaa0cbaec88b 0
#vwend

# 节点B：
[root@test2 mysql]# more gvwstate.dat
my_uuid: 969e273e-e072-11eb-98eb-eaa0cbaec88b
#vwbeg
view_id: 3 6c9f69d0-e072-11eb-80bf-ce66d30836b1 16
bootstrap: 0
member: 6c9f69d0-e072-11eb-80bf-ce66d30836b1 0
member: 969e273e-e072-11eb-98eb-eaa0cbaec88b 0
#vwend

# 节点C：
[root@test3 mysql]# more gvwstate.dat
my_uuid: 6c9f69d0-e072-11eb-80bf-ce66d30836b1
#vwbeg
view_id: 3 6c9f69d0-e072-11eb-80bf-ce66d30836b1 16
bootstrap: 0
member: 6c9f69d0-e072-11eb-80bf-ce66d30836b1 0
member: 969e273e-e072-11eb-98eb-eaa0cbaec88b 0
#vwend
```

文件释义如下：

```properties
my_uuid: 本节点的UUID
#vwbeg 视图信息开始标志
view_id: [view_type] [view_uuid] [view_seq]，view_type始终为3代表primary视图，view_seq表示视图操作序列号，值越大表示该视图的数据越新
bootstrap: 0或1，但不影响pc.recovery过程
member: [node's uuid] [node's segment]，表示节点关闭时的集群中所有正常的节点
member:
member:
#vwend 试图信息结束标志
```

通过查看该文件，可以得出与方法一同样的结论，即B、C节点的数据最新，可作为主节点启动集群。

​	上述示例中，节点A、B、C异常关闭的时间不同，且不停有数据写入，因此各个节点的 **wsrep** 的位置（操作序列号）不同，集群无法自动恢复。
​	如果发生例如数据中心断电、ABC节点同时异常关闭的情况，此时**ABC节点的wsrep位置相同，则恢复供电、ABC服务器启动后，集群可自动恢复，无需人工干预。**
​	**这也是 pc.recovery 特性的一个限制，即节点位置必须相同才可自动恢复。**

## 7、[集群因脑裂而失去其主要状态](https://docs.percona.com/percona-xtradb-cluster/8.0/crash-recovery.html#scenario-7-the-cluster-loses-its-primary-state-due-to-split-brain)

​	为了便于本示例，我们假设集群由偶数个节点组成：例如 6 个。其中三个节点位于一个位置，另外三个节点位于另一个位置，并且它们会断开网络连接。最佳做法是避免这种拓扑结构：如果实际节点数无法达到奇数，可以使用额外的**仲裁节点 (garbd)，或为某些节点设置更高的 pc.weight。**但是，一旦发生脑裂，任何分离的组都无法维持法定人数：所有节点都必须停止处理请求，并且集群的两个部分都将不断尝试重新连接。

​	如果您希望在网络链路恢复之前恢复服务，则可以使用与[场景 5：两个节点从集群中消失中所述的相同命令，将其中一个组再次设为主组](https://docs.percona.com/percona-xtradb-cluster/8.0/crash-recovery.html#scenario-5-two-nodes-disappear-from-the-cluster)

```
> SET GLOBAL wsrep_provider_options='pc.bootstrap=true';
```

此后，您就可以处理集群的手动恢复部分，而另一半应该能够在网络链接恢复后立即使用[IST自动重新加入。](https://docs.percona.com/percona-xtradb-cluster/8.0/glossary.html#ist)

### 参考文档：

- [官网Crash Recovery](https://docs.percona.com/percona-xtradb-cluster/8.0/crash-recovery.html)

- [Document gvwstate.dat · Issue #84 · codership/galera · GitHub](https://github.com/codership/galera/issues/84)
  ————————————————

原文链接：https://blog.csdn.net/sinat_32582203/article/details/118611017