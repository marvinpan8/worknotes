# [clickhouse基本数据类型总结](https://www.cnblogs.com/wshenjin/p/13072989.html)

原文：<https://www.cnblogs.com/wshenjin/p/13072989.html>

------

#### 整型

有符号整型（-2n-1~2n-1-1）：

- Int8 - [-128 : 127]
- Int16 - [-32768 : 32767]
- Int32 - [-2147483648 : 2147483647]
- Int64 - [-9223372036854775808 : 9223372036854775807]

无符号整型范围（0~2n-1）：

- UInt8 - [0 : 255]
- UInt16 - [0 : 65535]
- UInt32 - [0 : 4294967295]
- UInt64 - [0 : 18446744073709551615]

#### 浮点型

- Float32 - float
- Float64 – double

#### 布尔型

没有单独的类型来存储布尔值。可以使用 UInt8 类型，取值限制为 0 或 1。

#### 字符串

- 变长字符串 String
  字符串可以任意长度的。它可以包含任意的字节集，包含空字节。
- 定长字符串 FixedString(N)
  固定长度 N 的字符串，N 必须是严格的正自然数。当服务端读取长度小于 N 的字符串时候，通过在字符串末尾添加空字节来达到 N 字节长度。 当服务端读取长度大于 N 的字符串时候，将返回错误消息。
  与String相比，极少会使用FixedString，因为使用起来不是很方便。

#### 枚举类型

- Enum8 用 'String'= Int8 对描述。
- Enum16 用 'String'= Int16 对描述。

Enum 保存 'string'= integer 的对应关系。在 ClickHouse 中，尽管用户使用的是字符串常量，但所有含有 Enum 数据类型的操作都是按照包含整数的值来执行。这在性能方面比使用 String 数据类型更有效。
举例：

```shell
#新建一张带Enum8类型的表：
localhost :) CREATE TABLE  enum_t (  et Enum8('a' = 1, 'b' = 2, 'c' =3)) ENGINE = TinyLog;
localhost :) DESC enum_t ;
┌─name─┬─type─────────────────────────────┬─default_type─┬─default_expression─┬─comment─┬─codec_expression─┬─ttl_expression─┐
│ et   │ Enum8('a' = 1, 'b' = 2, 'c' = 3) │              │                    │         │                  │                │
└──────┴──────────────────────────────────┴──────────────┴────────────────────┴─────────┴──────────────────┴────────────────┘

#插入数据
localhost :) INSERT INTO enum_t(et) VALUES ('a'),('a'),('b');

#查看
localhost :) SELECT * FROM enum_t ;
┌─et─┐
│ a  │
│ a  │
│ b  │
└────┘

#如果需要看到对应行的数值，则必须将 Enum 值转换为整数类型：
localhost :) SELECT CAST(et, 'Int8') FROM enum_t ;
┌─CAST(et, 'Int8')─┐
│                1 │
│                1 │
│                2 │
└──────────────────┘
```

#### 数据组

- Array(T)

由 T 类型元素组成的数组。T 可以是任意类型，包含数组类型，但不推荐使用多维数组，ClickHouse 对多维数组的支持有限。
可以使用array()函数和中括号来创建数组

举例：

```shell
#新建两张带Array类型的表：
localhost :) CREATE TABLE array_t (arr Array(UInt8)) ENGINE = TinyLog;
localhost :) CREATE TABLE array_ta (arr Array(String)) ENGINE = TinyLog;

#插入数组
localhost :) INSERT INTO array_t VALUES([1,2,3,4,5]),(array(11,22,33,44,55));
localhost :) INSERT INTO array_ta VALUES(['a','b','c']),(array('x','y','z','123'));

#查看结果
localhost :) SELECT * FROM array_t ;
┌─arr──────────────┐
│ [1,2,3,4,5]      │
│ [11,22,33,44,55] │
└──────────────────┘

localhost :) SELECT * FROM array_ta;
┌─arr─────────────────┐
│ ['a','b','c']       │
│ ['x','y','z','123'] │
└─────────────────────┘
```

#### 元组

- Tuple(T1, T2, ...)

元组，其中每个元素都有单独的类型。

举个例子：

```shell
#创建一张带tuple字段的表：
localhost :) CREATE TABLE tuple_t (ttt Tuple(Int8, String, Array(String), Array(Int8))) ENGINE = TinyLog;

#插入数据
localhost :) INSERT INTO tuple_t VALUES((1, 'a', ['a', 'b', 'c'], [1, 2, 3])),(tuple(11, 'A', ['A', 'B', 'C'], [11, 22, 33]));

#查看数据
localhost :) SELECT * FROM tuple_t ;
┌─ttt───────────────────────────────┐
│ (1,'a',['a','b','c'],[1,2,3])     │
│ (11,'A',['A','B','C'],[11,22,33]) │
└───────────────────────────────────┘
```

#### 日期

- Date

用两个字节存储，表示从 1970-01-01 (无符号) 到当前的日期值, 最小值输出为0000-00-00。

#### 时间戳

- DateTime

用四个字节（无符号的）存储 Unix 时间戳，允许存储与日期类型相同的范围内的值。最小值为 0000-00-00 00:00:00，时间戳类型值精确到秒。

---
查询数据类型SQL

```bash
localhost :) SELECT * FROM system.data_type_families

┌─name────────────────────────────┬─case_insensitive─┬─alias_to────┐
│ Polygon                         │                0 │             │
│ Ring                            │                0 │             │
│ MultiPolygon                    │                0 │             │
│ IPv6                            │                0 │             │
│ IntervalSecond                  │                0 │             │
│ IPv4                            │                0 │             │
│ UInt32                          │                0 │             │
│ IntervalYear                    │                0 │             │
│ IntervalQuarter                 │                0 │             │
│ IntervalMonth                   │                0 │             │
│ Int64                           │                0 │             │
│ IntervalDay                     │                0 │             │
│ IntervalHour                    │                0 │             │
│ UInt256                         │                0 │             │
│ Int16                           │                0 │             │
│ LowCardinality                  │                0 │             │
│ AggregateFunction               │                0 │             │
│ Nothing                         │                0 │             │
│ Decimal256                      │                1 │             │
│ Tuple                           │                0 │             │
│ Array                           │                0 │             │
│ Enum16                          │                0 │             │
│ Enum8                           │                0 │             │
│ IntervalMinute                  │                0 │             │
│ FixedString                     │                0 │             │
│ String                          │                0 │             │
│ DateTime                        │                1 │             │
│ UUID                            │                0 │             │
│ Decimal64                       │                1 │             │
│ Nullable                        │                0 │             │
│ Enum                            │                0 │             │
│ Int32                           │                0 │             │
│ UInt8                           │                0 │             │
│ Date                            │                1 │             │
│ Decimal32                       │                1 │             │
│ Point                           │                0 │             │
│ Float64                         │                0 │             │
│ DateTime64                      │                1 │             │
│ Int128                          │                0 │             │
│ Decimal128                      │                1 │             │
│ Int8                            │                0 │             │
│ SimpleAggregateFunction         │                0 │             │
│ Nested                          │                0 │             │
│ Decimal                         │                1 │             │
│ Int256                          │                0 │             │
│ IntervalWeek                    │                0 │             │
│ UInt64                          │                0 │             │
│ UInt16                          │                0 │             │
│ Float32                         │                0 │             │
│ INET6                           │                1 │ IPv6        │
│ INET4                           │                1 │ IPv4        │
│ BINARY                          │                1 │ FixedString │
│ NATIONAL CHAR VARYING           │                1 │ String      │
│ BINARY VARYING                  │                1 │ String      │
│ NCHAR LARGE OBJECT              │                1 │ String      │
│ NATIONAL CHARACTER VARYING      │                1 │ String      │
│ NATIONAL CHARACTER LARGE OBJECT │                1 │ String      │
│ NATIONAL CHARACTER              │                1 │ String      │
│ NATIONAL CHAR                   │                1 │ String      │
│ CHARACTER VARYING               │                1 │ String      │
│ LONGBLOB                        │                1 │ String      │
│ MEDIUMTEXT                      │                1 │ String      │
│ TEXT                            │                1 │ String      │
│ TINYBLOB                        │                1 │ String      │
│ VARCHAR2                        │                1 │ String      │
│ CHARACTER LARGE OBJECT          │                1 │ String      │
│ DOUBLE PRECISION                │                1 │ Float64     │
│ LONGTEXT                        │                1 │ String      │
│ NVARCHAR                        │                1 │ String      │
│ INT1 UNSIGNED                   │                1 │ UInt8       │
│ VARCHAR                         │                1 │ String      │
│ CHAR VARYING                    │                1 │ String      │
│ MEDIUMBLOB                      │                1 │ String      │
│ NCHAR                           │                1 │ String      │
│ CHAR                            │                1 │ String      │
│ SMALLINT UNSIGNED               │                1 │ UInt16      │
│ TIMESTAMP                       │                1 │ DateTime    │
│ FIXED                           │                1 │ Decimal     │
│ TINYTEXT                        │                1 │ String      │
│ NUMERIC                         │                1 │ Decimal     │
│ DEC                             │                1 │ Decimal     │
│ TINYINT UNSIGNED                │                1 │ UInt8       │
│ INTEGER UNSIGNED                │                1 │ UInt32      │
│ INT UNSIGNED                    │                1 │ UInt32      │
│ CLOB                            │                1 │ String      │
│ MEDIUMINT UNSIGNED              │                1 │ UInt32      │
│ BOOL                            │                1 │ Int8        │
│ SMALLINT                        │                1 │ Int16       │
│ INTEGER SIGNED                  │                1 │ Int32       │
│ NCHAR VARYING                   │                1 │ String      │
│ INT SIGNED                      │                1 │ Int32       │
│ TINYINT SIGNED                  │                1 │ Int8        │
│ BIGINT SIGNED                   │                1 │ Int64       │
│ BINARY LARGE OBJECT             │                1 │ String      │
│ SMALLINT SIGNED                 │                1 │ Int16       │
│ MEDIUMINT                       │                1 │ Int32       │
│ INTEGER                         │                1 │ Int32       │
│ INT1 SIGNED                     │                1 │ Int8        │
│ BIGINT UNSIGNED                 │                1 │ UInt64      │
│ BYTEA                           │                1 │ String      │
│ INT                             │                1 │ Int32       │
│ SINGLE                          │                1 │ Float32     │
│ FLOAT                           │                1 │ Float32     │
│ MEDIUMINT SIGNED                │                1 │ Int32       │
│ BOOLEAN                         │                1 │ Int8        │
│ DOUBLE                          │                1 │ Float64     │
│ INT1                            │                1 │ Int8        │
│ CHAR LARGE OBJECT               │                1 │ String      │
│ TINYINT                         │                1 │ Int8        │
│ BIGINT                          │                1 │ Int64       │
│ CHARACTER                       │                1 │ String      │
│ BYTE                            │                1 │ Int8        │
│ BLOB                            │                1 │ String      │
│ REAL                            │                1 │ Float32     │
└─────────────────────────────────┴──────────────────┴─────────────┘
```

