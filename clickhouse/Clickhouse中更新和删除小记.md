# Clickhouse 数据删除更新

2020-06-20 https://blog.csdn.net/u010180815/article/details/106823138 

---

Clickhouse删除/更新数据（UPDATE/DELETE/DROP）与普通的sql语法有点不一样，因此做一下记录。

## 1 删除表
```sql
DROP table db.视图表 ON CLUSTER cluster_name;
DROP table db.本地表 ON CLUSTER cluster_name;
```
## 2 数据删除
按分区删除
```sql
ALTER TABLE db_name.table_name DROP PARTITION '20200601'
```
按条件删除
```sql
ALTER TABLE db_name.table_name DELETE WHERE day = '20200618'
```
## 3 数据更新
```sql
ALTER TABLE <table_name> UPDATE col1 = expr1, ... WHERE <filter>
```
> 注意：
> 1. 该命令必须在版本号大于1.1.54388才可以使用，适用于 mergeTree 引擎
> 2. 该命令是异步执行的，可以通过查看表 system.mutations 来查看命令的是否执行完毕

举例：
```sql
:) select event_status_key, count(*) from test_update where event_status_key in (0, 22) group by event_status_key;

┌─event_status_key─┬──count()─┐
│                0 │ 17824710 │
│               22 │     1701 │
└──────────────────┴──────────┘

:) ALTER TABLE test_update UPDATE event_status_key=0 where event_status_key=22;

0 rows in set. Elapsed: 0.067 sec.


:) select event_status_key, count(*) from test_update where event_status_key in (0, 22) group by event_status_key;

 ┌─event_status_key─┬──count()─┐
 │                0 │ 17826411 │
 └──────────────────┴──────────┘
```

## 4 Clickhouse更新操作有一些限制：
### ① 索引列不能进行更新
```sql
:) ALTER TABLE test_update UPDATE event_key = 41 WHERE event_key = 40;

Received exception from server (version 18.12.17):
Code: 420. DB::Exception: Received from localhost:9000, ::1. DB::Exception: Cannot UPDATE key column `event_key`.
```
### ② 分布式表不能进行更新
```sql
Received exception from server (version 18.12.17):
Code: 48. DB::Exception: Received from localhost:9000, ::1. DB::Exception: Mutations are not supported by storage Distributed.
```
**ALTER TABLE UPDATE/DELETE 不支持分布式DDL，因此需要在分布式环境中手动在每个节点上local的进行更新/删除数据。**

### ③ 不适合频繁更新或point更新

**由于Clickhouse更新操作非常耗资源，如果频繁的进行更新操作，可能会弄崩集群，请谨慎操作。**

参考：

https://www.altinity.com/blog/2018/10/16/updates-in-clickhouse
