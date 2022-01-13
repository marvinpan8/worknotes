# ClickHouse 中的更新和删除

https://altinity.com/blog/2018/10/16/updates-in-clickhouse 2018 年 10 月 17 日

---

ClickHouse 开发团队发布了一篇优秀的博客文章[“如何在 ClickHouse 中更新数据”](https://clickhouse.yandex/blog/en/how-to-update-data-in-clickhouse)已经两年了。在过去，ClickHouse 仅支持每月分区，对于可变数据结构，他们建议使用非常奇特的数据结构。我们都在等待更方便的方法，终于有了：ClickHouse 现在支持删除更新！在本文中，我们将看到它是如何工作的。

## 测试数据

让我们加载一个包含一些数据的测试表：

```sql
:) select count(*) from system.columns where table='test_update';

┌─count()─┐
│     332 │
└─────────┘

:) select count(*) from test_update;

┌──count()─┐
│ 17925050 │
└──────────┘
```

我们将尝试更新，因为它更有趣。删除工作几乎相同。

## 这个怎么运作

更新和删除的语法是非标准 SQL。ClickHouse 团队想表达与传统 SQL 的区别：新的更新和删除是批量操作，异步执行。它甚至被称为“突变”。自定义语法突出了差异。

```sql
ALTER TABLE <table_name> DELETE WHERE <filter>;
```

和

```sql
ALTER TABLE <table_name> UPDATE col1 = expr1, ... WHERE <filter>;
```

在我们的测试表中有列 event_status_key。

```sql
:) select event_status_key, count(*) from test_update where event_status_key in (0, 22) group by event_status_key;

┌─event_status_key─┬──count()─┐
│                0 │ 17824710 │
│               22 │     1701 │
└──────────────────┴──────────┘
```

让我们考虑一下状态 22 是一个错误，我们想要修复它。这只是数据的 0.01%，但如果没有 DELETE 或 UPDATE，我们将不得不重新加载表。

```sql
:) ALTER TABLE test_update UPDATE event_status_key=0 where event_status_key=22;

0 rows in set. Elapsed: 0.067 sec.
```

它立即返回，但更新是异步的，因此我们不知道数据是否已更新。让我们检查：

```sql
:) select event_status_key, count(*) from test_update where event_status_key in (0, 22) group by event_status_key;

 ┌─event_status_key─┬──count()─┐
 │                0 │ 17826411 │
 └──────────────────┴──────────┘
```

似乎正在工作。可以在 system.mutations 表中查找更新操作的状态：

```sql
:) select * from system.mutations where table='test_update';

Row 1:
──────
database:                   test
table:                      test_update
mutation_id:                mutation_162.txt
command:                    UPDATE event_status_key = 0 WHERE event_status_key = 22
create_time:                2018-10-12 12:39:32
block_numbers.partition_id: ['']
block_numbers.number:       [162]
parts_to_do:                0
is_done:                    1
```

另一个有趣的见解给出了 system.parts 表：

```sql
:) select name, active, rows, bytes_on_disk, modification_time from system.parts where table='test_update' order by modification_time;

┌─name──────────────┬─active─┬────rows─┬─bytes_on_disk─┬───modification_time─┐
│ all_1_36_2        │      0 │ 3841126 │     637611245 │ 2018-10-12 12:16:24 │
│ all_37_75_2       │      0 │ 4358144 │     598548358 │ 2018-10-12 12:16:47 │
│ all_112_117_1     │      0 │  638976 │     167899233 │ 2018-10-12 12:17:00 │
│ all_151_155_1     │      0 │  778240 │      27388052 │ 2018-10-12 12:17:29 │
│ all_76_111_2      │      0 │ 3833856 │     989762502 │ 2018-10-12 12:17:30 │
│ all_156_161_1     │      0 │  837460 │      27490891 │ 2018-10-12 12:17:43 │
│ all_118_150_2     │      0 │ 3637248 │     859673147 │ 2018-10-12 12:17:52 │
│ all_1_36_2_162    │      1 │ 3841126 │     637611232 │ 2018-10-12 12:39:32 │
│ all_37_75_2_162   │      1 │ 4358144 │     598548352 │ 2018-10-12 12:39:32 │
│ all_76_111_2_162  │      1 │ 3833856 │     989762502 │ 2018-10-12 12:39:32 │
│ all_112_117_1_162 │      1 │  638976 │     167899233 │ 2018-10-12 12:39:32 │
│ all_118_150_2_162 │      1 │ 3637248 │     859673147 │ 2018-10-12 12:39:32 │
│ all_151_155_1_162 │      1 │  778240 │      27388052 │ 2018-10-12 12:39:32 │
│ all_156_161_1_162 │      1 │  837460 │      27490891 │ 2018-10-12 12:39:32 │
└───────────────────┴────────┴─────────┴───────────────┴─────────────────────┘
```

它表明每个部分都被更新所触及。但对于小型测试数据集来说，它已经相当快了。

现在让我们尝试做一些更困难的事情。我们的表中有一个数组列保存整数段 ID。

```sql
:) select count(*) from test_update where has(dmp_audience_ids, 31694239);
┌─count()─┐
│  228706 │
└─────────┘
```

考虑到我们要向所有用户添加额外的细分市场，细分为 31694239。

```sql
:) alter table test_update update dmp_audience_ids = arrayPushBack(dmp_audience_ids, 1234567) where has(dmp_audience_ids, 31694239);
```

再次即时响应，让我们检查数据是否已正确更新。

```sql
:) select count(*) from test_update where has(dmp_audience_ids, 1234567)

┌─count()─┐
│  228706 │
└─────────┘

:) select dmp_audience_ids from test_update where has(dmp_audience_ids, 1234567) and length(dmp_audience_ids)<5 limit 1;

┌─dmp_audience_ids─────────────────────┐
│ [31694239,31694422,31694635,1234567] │
└──────────────────────────────────────┘
```

完美运行，速度非常快！

## 6 条评论

1. **Weitao Li** 说道：2019 年 3 月 21 日凌晨 2:55

   为什么clickhouse不支持分布式表的删除/更新？
   对于更新非主键列，为什么clickhouse 必须重建每个部分？

   1. **Alexander Zaitsev** 说：2019 年 3 月 28 日下午 4:59

      Hi Weitao，
      **由于更新和删除实际上是ALTER TABLE 语句，因此它们应用于本地表，而不是分布式表**。这些是您不应该经常进行的操作。如果你想在所有集群上都这样做，你可以使用分布式 ddl: ALTER TABLE ... ON CLUSTER
      对于更新，ClickHouse 只重建部分更新的列，这些列受 WHERE 条件的约束。在某些情况下，所有部件都会受到影响。但它不会重建所有列，只会重建更新的列。

      1. **Kiran**  说：2021 年 1 月 11 日上午 6:30

         仍然不适用于正在运行的 Alicoud Clickhouse ( <https://github.com/ClickHouse/ClickHouse/issues/9117> )

         e.displayText() = DB::Exception: Mutations are not supported by storage Distributed (version 20.3.10.75) 

2. **Lokesh Lal** 说：2019 年 5 月 20 日上午 6:00

   届时我们可以期待 clickhouse 为文章中提到的 UPDATE/DELETE 添加完整的 SQL 支持。谢谢

3. **James Dear** 说：2019 年 8 月 29 日上午 9:02

   您可以使用查询 VALUES 中的 where 条件删除吗？我的意思是这样的：client.execute("ALTER TABLE my_table DELETE WHERE uid IN VALUES", [(12345,), (23456,)])

   1. **Alexander Zaitsev**  说：2019 年 8 月 29 日下午 1:31

      Hi James，您不能使用 VALUES，但您可以使用 IN 列表，就像在普通 SQL 查询中一样，例如：
      “ALTER TABLE my_table DELETE WHERE uid IN (12345, 23456)”
      您使用的是什么客户端？