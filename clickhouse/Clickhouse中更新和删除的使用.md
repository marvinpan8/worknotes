# Clickhouse中update/delete的使用

https://www.jianshu.com/p/521f2d1611f8  2020.03.31

从使用场景来说，Clickhouse是个分析型数据库。这种场景下，数据一般是不变的，因此Clickhouse对update、delete的支持是比较弱的，实际上并不支持标准的update、delete操作。

下面介绍一下Clickhouse中update、delete的使用。

### 更新、删除语法

Clickhouse通过alter方式实现更新、删除，它把update、delete操作叫做mutation(突变)。语法为：

```sql
ALTER TABLE [db.]table DELETE WHERE filter_expr
ALTER TABLE [db.]table UPDATE column1 = expr1 [, ...] WHERE filter_expr
```

那么，mutation与标准的update、delete有什么区别呢？

标准SQL的更新、删除操作是同步的，即客户端要等服务端反回执行结果（通常是int值）；而Clickhouse的update、delete是通过异步方式实现的，当执行update语句时，服务端立即反回，但是实际上此时数据还没变，而是排队等着。

### 查看mutation队列

那么，怎么查看数据是否更新完成了呢？

可以通过system.mutations表查看相关信息：

```sql
SELECT
    database,
    table,
    command,
    create_time,
    is_done
FROM system.mutations
LIMIT 10

┌─database─┬─table─────────────────┬─command─────────────────────────────────────────────────────────────────────────────┬─────────create_time─┬─is_done─┐
│ app      │ scene_model           │ UPDATE status = '2' WHERE id = '208209306'                                          │ 2020-03-30 15:38:58 │       1 │
│ app      │ scene_model           │ UPDATE status = '2' WHERE id = '100000004'                                          │ 2020-03-30 15:40:00 │       1 │
│ app      │ scene_model           │ UPDATE status = '2' WHERE id = '100000004'                                          │ 2020-03-30 15:41:09 │       1 │
│ app      │ user_model            │ UPDATE name = 'zhuweiming' WHERE id = '0000000047fd31e40147fd3477cc0000'            │ 2020-03-19 18:34:59 │       1 │
│ app      │ work_statistics_total │ UPDATE pv = 10000, uv = 10000 WHERE (id = '1000900') AND (product = 'tracker_view') │ 2020-03-31 14:45:59 │       1 │
│ app      │ work_statistics_total │ UPDATE pv = 10000, uv = 10000 WHERE (id = '1000901') AND (product = 'tracker_view') │ 2020-03-31 14:45:59 │       1 │
│ app      │ work_statistics_total │ UPDATE pv = 10000, uv = 10000 WHERE (id = '1000902') AND (product = 'tracker_view') │ 2020-03-31 14:45:59 │       1 │
│ app      │ work_statistics_total │ UPDATE pv = 10000, uv = 10000 WHERE (id = '1000903') AND (product = 'tracker_view') │ 2020-03-31 14:45:59 │       1 │
│ app      │ work_statistics_total │ UPDATE pv = 10000, uv = 10000 WHERE (id = '1000904') AND (product = 'tracker_view') │ 2020-03-31 14:45:59 │       1 │
│ app      │ work_statistics_total │ UPDATE pv = 10000, uv = 10000 WHERE (id = '1000905') AND (product = 'tracker_view') │ 2020-03-31 14:45:59 │       1 │
└──────────┴───────────────────────┴─────────────────────────────────────────────────────────────────────────────────────┴─────────────────────┴─────────┘
```

- database: 库名
- table: 表名
- command: 更新/删除语句
- create_time: mutation任务创建时间，系统按这个时间顺序处理数据变更
- is_done: 是否完成，1为完成，0为未完成

除了上述的，还有一些其他的字段，详见：[官方文档](https://clickhouse.com/docs/zh/operations/system-tables/#system_tables-mutations)。

通过以上信息，可以查看当前有哪些mutation已经完成，is_done为1即表示已经完成。

### Mutation具体过程

首先，使用where条件找到需要修改的分区；
 然后，重建每个分区，用新的分区替换旧的，分区一旦被替换，就不可回退；

对于每个分区，可以认为是原子性的；但对于整个mutation，如果涉及多个分区，则不是原子性的。

### 注意事项

- 更新功能不支持更新有关主键或分区键的列
- 更新操作没有原子性，即在更新过程中select结果很可能是一部分变了，一部分没变，从上边的具体过程就可以知道
- 更新是按提交的顺序执行的
- 更新一旦提交，不能撤销，即使重启clickhouse服务，也会继续按照system.mutations的顺序继续执行
- 已完成更新的条目不会立即删除，保留条目的数量由finished_mutations_to_keep存储引擎参数确定。 超过数据量时旧的条目会被删除
- 更新可能会卡住，比如`update intvalue='abc'`这种类型错误的更新语句执行不过去，那么会一直卡在这里，此时，可以使用`KILL MUTATION`来取消，语法：

```bash
# database、table是system.mutations表中的字段
kill mutation where database='app' and table='test'
```

### 使用建议

按照官方的说明，update/delete 的使用场景是一次更新大量数据，也就是where条件筛选的结果应该是一大片数据。

举例：`alter table test update status=1 where status=0 and day='2020-04-01'`，一次更新一天的数据。

那么，能否一次只更新一条数据呢？例如：`alter table test update pv=110 where id=100`

当然也可以，但频繁的这种操作，可能会对服务造成压力。这很容易理解，如上文提到，更新的单位是分区，如果只更新一条数据，那么需要重建一个分区；如果更新100条数据，而这100条可能落在3个分区上，则需重建3个分区；相对来说一次更新一批数据的整体效率远高于一次更新一行。

对于频繁单条更新的这种场景，建议使用`ReplacingMergeTree`引擎来变相解决。具体如何使用，以后有时间再整理。