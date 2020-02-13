# MySQL 5.7 基于 GTID 的主从复制实践

*osted by Mike on 2017-07-03*  <https://www.hi-linux.com/posts/47176.html>

在 「[MySQL 5.7多源复制实践](https://www.hi-linux.com/posts/61083.html)」 一文中我们讲解了 MySQL 5.7 新特性多源复制的实现方法。今天我们来讲讲 MySQL 5.7 的另一个新特性基于 GTID 的主从复制实现。

### 什么是GTID Replication

从 MySQL 5.6.5 开始新增了一种基于 GTID 的复制方式。通过 GTID 保证了每个在主库上提交的事务在集群中有一个唯一的ID。这种方式强化了数据库的主备一致性，故障恢复以及容错能力。

在原来基于二进制日志的复制中，从库需要告知主库要从哪个偏移量进行增量同步，如果指定错误会造成数据的遗漏，从而造成数据的不一致。借助GTID，在发生主备切换的情况下，MySQL的其它从库可以自动在新主库上找到正确的复制位置，这大大简化了复杂复制拓扑下集群的维护，也减少了人为设置复制位置发生误操作的风险。另外，基于GTID的复制可以忽略已经执行过的事务，减少了数据发生不一致的风险。

**什么是GTID**

GTID (Global Transaction ID) 是对于一个已提交事务的编号，并且是一个全局唯一的编号。 GTID 实际上 是由 UUID+TID 组成的。其中 UUID 是一个 MySQL 实例的唯一标识。TID 代表了该实例上已经提交的事务数量，并且随着事务提交单调递增。下面是一个GTID的具体形式：

```properties
3E11FA47-71CA-11E1-9E33-C80AA9429562:23
```

一组连续的事务可以用 `-` 连接的事务序号范围表示。例如：

```properties
e6954592-8dba-11e6-af0e-fa163e1cf111:1-5
```

GTID 集合可以包含来自多个 MySQL 实例的事务，它们之间用逗号分隔。如果来自同一 MySQL 实例的事务序号有多个范围区间，各组范围之间用冒号分隔。例如：

```properties
e6954592-8dba-11e6-af0e-fa163e1cf111:1-5:11-18,
e6954592-8dba-11e6-af0e-fa163e1cf3f2:1-27
```

可以使用 `SHOW MASTER STATUS` 实时看当前的事务执行数。

**GTID的作用**

GTID 的使用不单单是用单独的标识符替换旧的二进制日志文件和位置，它也采用了新的复制协议。旧的协议往往简单直接，即：首先从服务器上在一个特定的偏移量位置连接到主服务器上一个给定的二进制日志文件，然后主服务器再从给定的连接点开始发送所有的事件。

新协议稍有不同：支持以全局统一事务 ID (GTID) 为基础的复制。当在主库上提交事务或者被从库应用时，可以定位和追踪每一个事务。GTID 复制是全部以事务为基础，使得检查主从一致性变得非常简单。如果所有主库上提交的事务也同样提交到从库上，一致性就得到了保证。

GTID 相关操作：默认情况下将一个事务记录进二进制文件时，首先记录它的 GTID，而且 GTID 和事务相关信息一并要发送给从服务器，由从服务器在本地应用认证，但是绝对不会改变原来的事务 ID 号。因此在 GTID 的架构上就算有了N层架构，复制是N级架构，事务 ID 依然不会改变，有效的保证了数据的完整和安全性。

你可以使用基于语句的或基于行的复制与 GTID ，但是，为了获得最佳效果，我们建议你使用基于行（ROW）的格式。

GTID功能的具体归纳主要有以下两点：

- 根据 GTID 可以知道事务最初是在哪个实例上提交的
- GTID 的存在方便了 Replication 的 Failover

我们可以看下在 MySQL 5.6 的 GTID 出现以前 Replication Failover 的操作过程。假设我们有一个如下图的环境：

![img](assets/mysql-gtid-3.png)

此时，Server A 的服务器宕机，需要将业务切换到 Server B 上。同时，我们又需要将 Server C 的复制源改成 Server B。复制源修改的命令语法很简单即：

```mysql
CHANGE MASTER TO MASTER_HOST='xxx', MASTER_LOG_FILE='xxx', MASTER_LOG_POS=nnnn
```

而这种方式的难点在于，由于同一个事务在每台机器上所在的 binlog 名字和位置都不一样，那么怎么找到 Server C 当前同步停止点对应 Server B 上 `master_log_file` 和 `master_log_pos` 的位置就成为了难题。

这也就是为什么 `M-S` 复制集群需要使用 `MMM`、`MHA` 这样的额外管理工具的一个重要原因。 这个问题在 5.6 的 GTID 出现后，就显得非常的简单。

由于同一事务的 GTID 在所有节点上的值一致，那么根据 Server C 当前停止点的 GTID 就能唯一定位到 Server B 上的GTID。甚至由于 `MASTER_AUTO_POSITION` 功能的出现，我们都不需要知道 GTID 的具体值。直接使用以下命令就可以直接完成 Failover 的工作。

```mysql
CHANGE MASTER TO MASTER_HOST='xxx', MASTER_AUTO_POSITION=='xxx'
```

**如何产生GTID**

GTID 的生成受 `gtid_next` 控制。 在主服务器上`gtid_next` 是默认的 `AUTOMATIC`，即在每次事务提交时自动生成新的 GTID 。它从当前已执行的 GTID 集合(即 `gtid_executed` )中，找一个大于 0 的未使用的最小值作为下个事务 GTID 。同时在 Binlog 的实际的更新事务事件前面插入一条 `set gtid_next` 事件。

这里以一条 insert 语句来看看 GTID 的生成过程：

![img](assets/mysql-gtid-5.png)

在从库上回放主库的 Binlog 时，先执行 `SET @@SESSION.GTID_NEXT`语句，然后再执行 insert 语句，确保在主和备上这条 insert 对应于相同的 GTID。

一般情况下，GTID集合是连续的，但使用多线程复制(MTS)以及通过 `gtid_next` 进行人工干预时会导致 GTID 空洞。

![img](assets/mysql-gtid-6.png)

继续执行事务，MySQL 会分配一个最小的未使用 GTID，也就是从出现空洞的地方分配 GTID，最终会把空洞填上。

![img](assets/mysql-gtid-7.png)

这意味着严格来说我们即不能假设 GTID 集合是连续的，也不能假定 GTID 序号大的事务在GTID序号小的事务之后执行，事务的顺序应由事务记录在 Binlog 中的先后顺序决定。

**什么是server-uuid**

MySQL 5.6 以后用 128 位的 `server-uuid` 代替了原本的 32 位 `server-id` 的大部分功能。原因很简单，`server-id` 依赖于 my.cnf 的手工配置，有可能产生冲突。而自动产生 128 位 UUID 的算法可以保证所有的 MySQL UUID 都不会冲突。

MySQL 在数据目录下有一个 auto.cnf 文件就是用来保存 `server-uuid` 的，如下：

```bash
$ cat /var/lib/mysql/auto.cnf
[auto]
server-uuid=f75ae43f-3f5e-11e7-9b98-001c4297532a
```

在 MySQL 再次启动时会读取 auto.cnf 文件，继续使用上次生成的 `server_uuid`。使用 `SHOW` 命令可以查看 MySQL 实例当前使用的 `server-uuid`，它是一个 MySQL Global Variables。

```mysql
SHOW GLOBAL VARIABLES LIKE ‘server_uuid';
```

### 配置GTID主从复制

#### 环境准备

这里一共使用了二台机器，MySQL 版本都为 5.7.18。

| 机器名        | IP地址        | MySQL角色  |
| ------------- | ------------- | ---------- |
| dev-master-01 | 192.168.2.210 | MySQL 主库 |
| dev-node-02   | 192.168.2.212 | MySQL 从库 |

#### 安装MySQL

MySQL 安装比较简单，在 「[MySQL 5.7多源复制实践](https://www.hi-linux.com/posts/61083.html)」一文中我们也讲了，这里就不在重复讲了。如果你还不会安装，可以先参考此文安装好 MySQL 。

#### 配置MySQL基于GTID的复制

GTID主从复制的配置思路

![img](assets/mysql-gtid-4.jpg)

- 修改MySQL主配置文件

配置 MySQL 基于GTID的复制，主要是需要在 MySQL 服务器的主配置文件 `[mysqld]` 段中添加以下内容：

```properties
gtid-mode = ON
enforce-gtid-consistency = ON
log-slave-updates = ON
```

在 MySQL 5.6 版本时，基于 GTID 的复制中 `log-slave-updates` 选项是必须的。但是其增大了从服务器的IO负载, 而在 MySQL 5.7 中该选项已经不是必须项。

MySQL主服务器配置片断

```bash
$ vim /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]

server-id = 1
binlog_format = row
expire_logs_days = 30
max_binlog_size  = 100M
gtid_mode = ON
enforce_gtid_consistency = ON
binlog-checksum = CRC32
master-verify-checksum = 1
log-bin = /var/log/mysql/mysql-bin
log_bin_index = /var/log/mysql/mysql-bin.index
log-slave-updates = ON
```

MySQL从服务器配置片断

```bash
$ vim /etc/mysql/mysql.conf.d/mysqld.cnf

[mysqld]

server-id = 3
gtid_mode = ON
enforce_gtid_consistency = ON
log-slave-updates = ON
skip-slave-start = true
expire_logs_days = 30
max_binlog_size  = 100M
read_only = ON
slave-sql-verify-checksum = 1
log-bin = /var/log/mysql/mysql-bin
log_bin_index = /var/log/mysql/mysql-bin.index
relay-log = /var/log/mysql/relay-log
relay-log-index = /var/log/mysql/relay-log-index
relay-log-info-file = /var/log/mysql/relay-log.info
master-info-repository = table
relay-log-info-repository = table
relay-log-recovery = ON
report-port = 3306
report-host = 192.168.2.212
replicate-do-db = master1
replicate_wild_do_table=master1.%
```

注：`server-id` 每台必须配置为不一样，比如 dev-master-01 为1，dev-node-02 为3。这里没有给出全部配置，其它请根据实际情况自行配置。

- 重启MySQL服务器

```bash
$ service mysql restart
```

- 创建具有复制权限的用户

基于 GTID 的复制会自动地将没有在从库执行过的事务重放，所以不要在其它从库上建立相同的账号。 如果建立了相同的账户，有可能造成复制链路的错误。

```mysql
# 在MySQL主服务器上创建
mysql> grant replication slave on *.* to 'repl'@'192.168.2.%' identified by '000000';
mysql> flush privileges;
```

- 查看主库与从库的GTID是否开启

```mysql
mysql> show variables like "%gtid%";

+----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| binlog_gtid_simple_recovery      | ON        |
| enforce_gtid_consistency         | ON        |
| gtid_executed_compression_period | 1000      |
| gtid_mode                        | ON        |
| gtid_next                        | AUTOMATIC |
| gtid_owned                       |           |
| gtid_purged                      |           |
| session_track_gtids              | OFF       |
+----------------------------------+-----------+
8 rows in set (0.00 sec)


mysql> show variables like '%gtid_next%';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| gtid_next     | AUTOMATIC |
+---------------+-----------+
1 row in set (0.00 sec)
```

简单说下几个常用参数的作用：

a) gtid_executed

在当前实例上执行过的 GTID 集合，实际上包含了所有记录到 binlog 中的事务。设置 `set sql_log_bin=0` 后执行的事务不会生成 binlog 事件，也不会被记录到 `gtid_executed` 中。执行 `RESET MASTER` 可以将该变量置空。

b) gtid_purged

binlog 不可能永远驻留在服务上，需要定期进行清理(通过 `expire_logs_days` 可以控制定期清理间隔)，否则迟早它会把磁盘用尽。

`gtid_purged` 用于记录本机上已经执行过，但是已经被清除了的 binlog 事务集合。它是 `gtid_executed` 的子集。只有 `gtid_executed` 为空时才能手动设置该变量，此时会同时更新 `gtid_executed` 为和 `gtid_purged` 相同的值。

`gtid_executed` 为空意味着要么之前没有启动过基于 GTID 的复制，要么执行过 `RESET MASTER`。执行 `RESET MASTER` 时同样也会把 `gtid_purged` 置空，即始终保持 `gtid_purged` 是 `gtid_executed`的子集。

c) gtid_next

会话级变量，指示如何产生下一个GTID。可能的取值如下:

第一个：AUTOMATIC

自动生成下一个 GTID，实现上是分配一个当前实例上尚未执行过的序号最小的 GTID。

第二个：ANONYMOUS

设置后执行事务不会产生GTID。

第三个：显式指定的GTID

可以指定任意形式合法的 GTID 值，但不能是当前 `gtid_executed` 中的已经包含的 GTID，否则下次执行事务时会报错。

- 查看服务器server_uuid

```mysql
mysql> show global variables like '%uuid%';

+---------------+--------------------------------------+
| Variable_name | Value                                |
+---------------+--------------------------------------+
| server_uuid   | f75ae43f-3f5e-11e7-9b98-001c4297532a |
+---------------+--------------------------------------+
1 row in set (0.00 sec)
```

- 查看主服务器状态

![img](assets/mysql-gtid-2.png)

- 从库连接至主库

```mysql
mysql> CHANGE MASTER TO MASTER_HOST='192.168.2.210',MASTER_USER='repl',MASTER_PASSWORD='000000',MASTER_AUTO_POSITION=1;
```

- 在从服务器上启动复制

```mysql
mysql> START SLAVE;
```

启动成功后查看SLAVE的状态

```mysql
mysql> SHOW SLAVE STATUS\G

...
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
...
```

确认 `Slave_IO_Running` 和 `Slave_SQL_Running` 两个参数都为 Yes 状态。

在主服务器查看从库连接的主机信息

![img](assets/mysql-gtid-1.png)

#### 测试GTID主从复制

- 在主库(dev-master-01)实例创建一些数据。

```mysql
mysql> create database master1;
mysql> use master1;
mysql> CREATE TABLE `test1` (`id` int(11) DEFAULT NULL,`count` int(11) DEFAULT NULL);
mysql> insert into test1 values(1,1);
```

- 在从库(dev-node-02)实例检查数据是否成功复制。

```mysql
mysql> select * from master1.test1;
+------+-------+
| id   | count |
+------+-------+
|    1 |     1 |
+------+-------+
1 row in set (0.00 sec)
```

检查从服务器状态

```mysql
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.2.210
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 977
               Relay_Log_File: relay-log.000002
                Relay_Log_Pos: 1190
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: master1
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table: master1.%
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 977
              Relay_Log_Space: 1391
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: f75ae43f-3f5e-11e7-9b98-001c4297532a
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: f75ae43f-3f5e-11e7-9b98-001c4297532a:1-4
            Executed_Gtid_Set: 2c55f623-4fea-11e7-82c7-001c4283459b:1,
f75ae43f-3f5e-11e7-9b98-001c4297532a:1-4
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

可以看到 IO 和 SQL 线程都为 YES ，另外 `retrieved_Gtid_Set` 接收了4个事务，`Executed_Gtid_Set` 执行了4个事务。

- 如何修复GTID复制错误

在基于 GTID 的复制拓扑中，要想修复从库的 SQL 线程错误，过去的 `SQL_SLAVE_SKIP_COUNTER` 方式不再适用。需要通过设置 `gtid_next` 或 `gtid_purged` 来完成，当然前提是已经确保主从数据一致，仅仅需要跳过复制错误让复制继续下去。

在从库上执行以下SQL：

```mysql
mysql> stop slave;
Query OK, 0 rows affected (0.00 sec)

mysql> set gtid_next='f75ae43f-3f5e-11e7-9b98-001c4297532a:20';
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> set gtid_next='AUTOMATIC';
Query OK, 0 rows affected (0.00 sec)

mysql> start slave;
Query OK, 0 rows affected (0.02 sec)
```

其中 `gtid_next` 就是跳过某个执行事务，设置 `gtid_next` 的方法一次只能跳过一个事务，要批量的跳过事务可以通过设置 `gtid_purged` 完成。假设下面的场景：

![img](assets/mysql-gtid-8.png)

此时从库的 `Executed_Gtid_Set` 已经包含了主库上 1-13 和 20 的事务，再开启复制会从后面的事务开始执行，就不会出错了。在从库上验证刚才插入的数据：

![img](assets/mysql-gtid-9.png)

注意，使用 `gtid_next` 和 `gtid_purged` 修复复制错误的前提是跳过那些事务后仍可以确保主备数据一致。如果做不到，就要考虑 pt-table-sync 或者重新导入备份的方式了。

#### GTID与备份恢复

在做备份恢复的时候，有时需要恢复出来的 MySQL 实例可以作为从库连上原来的主库继续复制，这就要求从备份恢复出来的 MySQL 实例拥有和主数据库数据一致的 `gtid_executed` 值。这也是通过设置 `gtid_purged`实现的，下面看下 mysqldump 做备份的例子。

- 通过mysqldump在主库上做一个全量备份

这里使用 `--all-databases`选项是因为基于 GTID 的复制会记录全部的事务, 所以要构建一个完整的dump。

```bash
$ mysqldump --all-databases --single-transaction  --triggers --routines --events --host=127.0.0.1 --port=3306 --user=root -p000000 > dump.sql
```

生成的 dump.sql 文件里包含了设置 `gtid_purged` 的语句

```mysql
SET @MYSQLDUMP_TEMP_LOG_BIN = @@SESSION.SQL_LOG_BIN;
SET @@SESSION.SQL_LOG_BIN= 0;
...
SET @@GLOBAL.GTID_PURGED='f75ae43f-3f5e-11e7-9b98-001c4297532a:1-14:20';
...
SET @@SESSION.SQL_LOG_BIN = @MYSQLDUMP_TEMP_LOG_BIN;
```

在从库恢复数据前需要先通过 `reset master` 清空 `gtid_executed` 变量

```bash
$ mysql -h127.0.0.1 --user=root -p000000 -e  'reset master'
$ mysql -h127.0.0.1 --user=root -p000000<dump.sql
```

否则执行设置 `GTID_PURGED` 的 SQL 时会报下面的错误：

```mysql
ERROR 1840 (HY000) at line 24: @@GLOBAL.GTID_PURGED can only be set when @@GLOBAL.GTID_EXECUTED is empty.
```

此时恢复出的 MySQL 实例的 `GTID_EXECUTED` 和在主库备份时的一致：

![img](assets/mysql-gtid-10.png)

由于恢复出 MySQL 实例已经被设置了正确的 `GTID_EXECUTED` ，下面以 `master_auto_postion = 1`的方式 `CHANGE MASTER` 到原来的主节点即可开始复制。

```mysql
mysql> CHANGE MASTER TO MASTER_HOST='192.168.2.210', MASTER_USER='repl', MASTER_PASSWORD='000000', MASTER_AUTO_POSITION = 1;
```

如果不希望备份文件中生成设置 `GTID_PURGED` 的 SQL，可以给 `mysqldump`传入 `--set-gtid-purged=OFF` 关闭。

### 其它一些需要注意的点

`enforce_gtid_consistency` 强制 GTID 一致性, 启用后以下命令无法再使用。

- create table … select …

```mysql
mysql> create table test2 select * from test1;
ERROR 1786 (HY000): Statement violates GTID consistency: CREATE TABLE ... SELECT.
```

因为实际上是两个独立事件，所以只能将其拆分。先建立表，然后再把数据插入到表中。

- 事务内部不能创建临时表

```mysql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> create temporary table test2(id int);
ERROR 1787 (HY000): Statement violates GTID consistency: CREATE TEMPORARY TABLE and DROP TEMPORARY TABLE can only be executed outside transactional context.  These statements are also not allowed in a function or trigger because functions and triggers are also considered to be multi-statement transactions.
```

- 同一事务中不能同时更新事务表与非事务表(MyISAM)，建议都选择 Innodb 作为默认的数据库引擎。

```mysql
mysql> CREATE TABLE `test_innodb` (id INT(11) UNSIGNED NOT NULL PRIMARY KEY AUTO_INCREMENT);
Query OK, 0 rows affected (0.04 sec)

mysql> CREATE TABLE `test_myisam` (id INT(11) UNSIGNED NOT NULL PRIMARY KEY AUTO_INCREMENT)   ENGINE = `MyISAM`;
Query OK, 0 rows affected (0.03 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test_innodb(id) value(1);
Query OK, 1 row affected (0.00 sec)

mysql> insert into test_myisam(id) value(1);
ERROR 1785 (HY000): Statement violates GTID consistency: Updates to non-transactional tables can only be done in either autocommitted statements or single-statement transactions, and never in the same statement as updates to transactional tables.
```

### 参考文档

[http://www.google.com](http://www.google.com/)
<http://www.ywnds.com/?p=3898>
<http://dbaplus.cn/news-11-857-1.html>
<http://www.jianshu.com/p/3675fa74bc72>
<http://www.cnblogs.com/abobo/p/4242417.html>
<http://cenalulu.github.io/mysql/mysql-5-6-gtid-basic/>