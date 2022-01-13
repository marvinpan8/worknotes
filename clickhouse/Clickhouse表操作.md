# Clickhouse 表操作

原文： https://blog.csdn.net/vkingnew/article/details/107309073

## 运行环境
```bash
$ cat /etc/centos-release
CentOS Linux release 7.6.1810 (Core) 

Clickhouse> select version();

SELECT version()

┌─version()─┐
│ 20.5.2.7  │
└───────────┘

1 rows in set. Elapsed: 0.001 sec. 
```
## 表的定义

### 常规表的定义：
```bash
Clickhouse> create table scott.emp(empno int not null comment '员工编码',ename varchar(32) comment '员工姓名',job varchar(32) comment '职位',mgr_no int  comment '领导的员工编号',hiredate Date comment '入职日期',sal decimal(7,2) comment '月薪',comm decimal(7,2) comment '奖金') engine=MergeTree() order by empno;

CREATE TABLE scott.emp
(
    `empno` int NOT NULL COMMENT '员工编码',
    `ename` varchar(32) COMMENT '员工姓名',
    `job` varchar(32) COMMENT '职位',
    `mgr_no` int COMMENT '领导的员工编号',
    `hiredate` Date COMMENT '入职日期',
    `sal` decimal(7, 2) COMMENT '月薪',
    `comm` decimal(7, 2) COMMENT '奖金'
)
ENGINE = MergeTree()
ORDER BY empno
```

### 添加字段：
```bash
Clickhouse> alter table scott.emp add column createtime datetime  default now() comment '数据写入时间';

ALTER TABLE scott.emp
    ADD COLUMN `createtime` datetime DEFAULT now() COMMENT '数据写入时间'

Ok.

0 rows in set. Elapsed: 0.014 sec. 
```

### 查看表的结构：
```bash
Clickhouse> desc scott.emp;

DESCRIBE TABLE scott.emp

┌─name───────┬─type──────────┬─default_type─┬─default_expression─┬─comment────────┬─codec_expression─┬─ttl_expression─┐
│ empno      │ Int32         │              │                    │ 员工编码       │                  │                │
│ ename      │ String        │              │                    │ 员工姓名       │                  │                │
│ job        │ String        │              │                    │ 职位           │                  │                │
│ mgr_no     │ Int32         │              │                    │ 领导的员工编号 │                  │                │
│ hiredate   │ Date          │              │                    │ 入职日期       │                  │                │
│ sal        │ Decimal(7, 2) │              │                    │ 月薪           │                  │                │
│ comm       │ Decimal(7, 2) │              │                    │ 奖金           │                  │                │
│ createtime │ DateTime      │ DEFAULT      │ now()              │ 数据写入时间   │                  │                │
└────────────┴───────────────┴──────────────┴────────────────────┴────────────────┴──────────────────┴────────────────┘

8 rows in set. Elapsed: 0.009 sec. 
```

### 查看表的定义：
```bash
Clickhouse> show create table scott.emp\G

SHOW CREATE TABLE scott.emp

Row 1:
──────
statement: CREATE TABLE scott.emp
(
    `empno` Int32 COMMENT '员工编码',
    `ename` String COMMENT '员工姓名',
    `job` String COMMENT '职位',
    `mgr_no` Int32 COMMENT '领导的员工编号',
    `hiredate` Date COMMENT '入职日期',
    `sal` Decimal(7, 2) COMMENT '月薪',
    `comm` Decimal(7, 2) COMMENT '奖金',
    `createtime` DateTime DEFAULT now() COMMENT '数据写入时间'
)
ENGINE = MergeTree()
ORDER BY empno
SETTINGS index_granularity = 8192

1 rows in set. Elapsed: 0.001 sec. 
```

## 表的操作

### 追加字段：
```bash
Clickhouse> alter table scott.emp add column deptno int default 10 comment '部门编号';

ALTER TABLE scott.emp
    ADD COLUMN `deptno` int DEFAULT 10 COMMENT '部门编号'
```
或者通过after 关键字在指定字段后添加新的字段：
```bash
Clickhouse> alter table scott.emp add column updatetime datetime default now() after createtime ;

ALTER TABLE scott.emp
    ADD COLUMN `updatetime` datetime DEFAULT now()     AFTER createtime
```

### 添加注释：
```bash
Clickhouse> alter table scott.emp comment column updatetime '末次修改时间';

ALTER TABLE scott.emp
    COMMENT COLUMN updatetime '末次修改时间'
```
## 修改：

###  修改字段类型：
```bash
Clickhouse> alter table scott.emp modify column hiredate datetime;

ALTER TABLE scott.emp
    MODIFY COLUMN `hiredate` datetime
```

### 修改默认值：
```bash
Clickhouse> desc scott.emp;

DESCRIBE TABLE scott.emp

┌─name───────┬─type──────────┬─default_type─┬─default_expression─┬─comment────────┬─codec_expression─┬─ttl_expression─┐
│ empno      │ Int32         │              │                    │ 员工编码       │                  │                │
│ ename      │ String        │              │                    │ 员工姓名       │                  │                │
│ job        │ String        │              │                    │ 职位           │                  │                │
│ mgr_no     │ Int32         │              │                    │ 领导的员工编号 │                  │                │
│ hiredate   │ DateTime      │              │                    │ 入职日期       │                  │                │
│ sal        │ Decimal(7, 2) │              │                    │ 月薪           │                  │                │
│ comm       │ Decimal(7, 2) │              │                    │ 奖金           │                  │                │
│ createtime │ DateTime      │ DEFAULT      │ now()              │ 数据写入时间   │                  │                │
│ updatetime │ DateTime      │ DEFAULT      │ now()              │ 末次修改时间   │                  │                │
│ deptno     │ Int32         │ DEFAULT      │ 10                 │ 部门编号       │                  │                │
└────────────┴───────────────┴──────────────┴────────────────────┴────────────────┴──────────────────┴────────────────┘

10 rows in set. Elapsed: 0.001 sec. 
```
将deptno的默认值由10 修改为20：
```bash
Clickhouse> alter table scott.emp modify column deptno default 20;

ALTER TABLE scott.emp
    MODIFY COLUMN `deptno` DEFAULT 20


Ok.

0 rows in set. Elapsed: 0.003 sec. 
```
查看表
```bash
Clickhouse> desc scott.emp;

DESCRIBE TABLE scott.emp

┌─name───────┬─type──────────┬─default_type─┬─default_expression─┬─comment────────┬─codec_expression─┬─ttl_expression─┐
│ empno      │ Int32         │              │                    │ 员工编码       │                  │                │
│ ename      │ String        │              │                    │ 员工姓名       │                  │                │
│ job        │ String        │              │                    │ 职位           │                  │                │
│ mgr_no     │ Int32         │              │                    │ 领导的员工编号 │                  │                │
│ hiredate   │ DateTime      │              │                    │ 入职日期       │                  │                │
│ sal        │ Decimal(7, 2) │              │                    │ 月薪           │                  │                │
│ comm       │ Decimal(7, 2) │              │                    │ 奖金           │                  │                │
│ createtime │ DateTime      │ DEFAULT      │ now()              │ 数据写入时间   │                  │                │
│ updatetime │ DateTime      │ DEFAULT      │ now()              │ 末次修改时间   │                  │                │
│ deptno     │ Int32         │ DEFAULT      │ 20                 │ 部门编号       │                  │                │
└────────────┴───────────────┴──────────────┴────────────────────┴────────────────┴──────────────────┴────────────────┘

10 rows in set. Elapsed: 0.002 sec. 
```
可以看到默认值已经修改。

### 修改TTL的信息：
```bash
Clickhouse> alter table scott.emp add column remark varchar(128) comment '说明信息' TTL createtime + toIntervalDay(31);

ALTER TABLE scott.emp
    ADD COLUMN `remark` varchar(128) COMMENT '说明信息' TTL createtime + toIntervalDay(31)


Ok.

0 rows in set. Elapsed: 0.012 sec. 
```
修改保存为62天：
```bash
Clickhouse> alter table scott.emp modify column remark varchar(254) TTL createtime+ toIntervalDay(62);

ALTER TABLE scott.emp
    MODIFY COLUMN `remark` varchar(254) TTL createtime + toIntervalDay(62)


Ok.

0 rows in set. Elapsed: 0.004 sec. 
```
可以查看表结构信息：
```bash
Clickhouse> desc scott.emp;

DESCRIBE TABLE scott.emp

┌─name───────┬─type──────────┬─default_type─┬─default_expression─┬─comment────────┬─codec_expression─┬─ttl_expression─────────────────┐
│ empno      │ Int32         │              │                    │ 员工编码       │                  │                                │
│ ename      │ String        │              │                    │ 员工姓名       │                  │                                │
│ job        │ String        │              │                    │ 职位           │                  │                                │
│ mgr_no     │ Int32         │              │                    │ 领导的员工编号 │                  │                                │
│ hiredate   │ DateTime      │              │                    │ 入职日期       │                  │                                │
│ sal        │ Decimal(7, 2) │              │                    │ 月薪           │                  │                                │
│ comm       │ Decimal(7, 2) │              │                    │ 奖金           │                  │                                │
│ createtime │ DateTime      │ DEFAULT      │ now()              │ 数据写入时间   │                  │                                │
│ updatetime │ DateTime      │ DEFAULT      │ now()              │ 末次修改时间   │                  │                                │
│ deptno     │ Int32         │ DEFAULT      │ 20                 │ 部门编号       │                  │                                │
│ remark     │ String        │              │                    │ 说明信息       │                  │ createtime + toIntervalDay(62) │
└────────────┴───────────────┴──────────────┴────────────────────┴────────────────┴──────────────────┴────────────────────────────────┘

11 rows in set. Elapsed: 0.002 sec. 
```
### 删除字段：
```bash
Clickhouse> alter table scott.emp drop column remark;

ALTER TABLE scott.emp
    DROP COLUMN remark
```
### 表的重命名：
```bash
create table default.dept  
(
    `deptno` Int32,
    `dname` String,
    `loc` String
)
ENGINE = MergeTree()
ORDER BY deptno;
```
可以将default.dept  ---> scott.dept:

```bash
Clickhouse> rename table default.dept to scott.dept;

RENAME TABLE default.dept TO scott.dept

Ok.
```

```bash
Clickhouse> rename table scott.dept  to scott.department;

RENAME TABLE scott.dept TO scott.department

Ok.

0 rows in set. Elapsed: 0.002 sec. 
```
表的重命名智能在单个节点范围之内运行，即只能在同一服务节点之内，不能在集群中的远程节点。

### 清空表的数据：
```bash
truncate table scott.department;
```

### 复制表的结构：
```bash
Clickhouse> create table if not exists t_emp as scott.emp engine=TinyLog;

CREATE TABLE IF NOT EXISTS t_emp AS scott.emp
ENGINE = TinyLog

Ok.

0 rows in set. Elapsed: 0.003 sec. 
```

### 复制表结构和数据：
```bash
Clickhouse> create table if not exists t_employee engine=Memory as select * from scott.emp;

CREATE TABLE IF NOT EXISTS t_employee
ENGINE = Memory AS
SELECT *
FROM scott.emp

Ok.

0 rows in set. Elapsed: 0.011 sec. 
```

### 表的字段重命名：(20.4.2+版本支持)
```bash
Clickhouse> alter table t_city rename column city_level TO  cityLevel;

ALTER TABLE t_city
    RENAME COLUMN city_level TO cityLevel


Ok.

0 rows in set. Elapsed: 0.018 sec. 
```
### null字段的修改：
```bash
Clickhouse> create table t(id int ,name varchar(32)) ENGINE = MergeTree PARTITION BY id ORDER BY id;

CREATE TABLE t
(
    `id` int,
    `name` varchar(32)
)
ENGINE = MergeTree
PARTITION BY id
ORDER BY id
```
插入数据
```bash
Clickhouse> insert into t(id,name)values(3,null);

INSERT INTO t (id, name) VALUES


Exception on client:
Code: 53. DB::Exception: Cannot insert NULL value into a column of type 'String' at: null);

Clickhouse> insert into t(id,name)values(1,'wuhan');

Clickhouse> insert into t(id)values(2);
```

修改表的定义：
```bash
Clickhouse> alter table t modify column name Nullable(varchar(32));

ALTER TABLE t
    MODIFY COLUMN `name` Nullable(varchar(32))


Ok.

0 rows in set. Elapsed: 0.017 sec. 
```
在此插入：
```bash
Clickhouse> insert into t(id,name)values(3,null);
Clickhouse> select * from t order by id FORMAT PrettyCompactMonoBlock;

SELECT *
FROM t
ORDER BY id ASC
FORMAT PrettyCompactMonoBlock

┌─id─┬─name──┐
│  1 │ wuhan │
│  2 │       │
│  3 │ ᴺᵁᴸᴸ  │
└────┴───────┘

3 rows in set. Elapsed: 0.002 sec. 
```
将null字段修改非null字段：
```bash
Code: 349. DB::Exception: Received from localhost:9000. DB::Exception: Cannot convert NULL value to non-Nullable type: (while reading from part /var/lib/clickhouse/data/default/t/3_4_4_0/): While executing MergeTreeThread. 

2 rows in set. Elapsed: 0.105 sec. 
```

查询的时候报错：

```bash
Clickhouse> create table t1(id Nullable(int),name Nullable(String)) engine=MergeTree() order by id;

CREATE TABLE t1
(
    `id` Nullable(int),
    `name` Nullable(String)
)
ENGINE = MergeTree()
ORDER BY id


Received exception from server (version 20.5.2):
Code: 44. DB::Exception: Received from localhost:9000. DB::Exception: Sorting key cannot contain nullable columns. 

0 rows in set. Elapsed: 0.011 sec. 
```

### 结论：
1.可以将非null字段修改为null字段，有了数据之后就不能修改会非null.
2.null 字段不能在MergeTree系列表引擎中作为order by 字段

## 临时表：

临时表的创建语法：
```bash
CREATE TEMPORARY TABLE [IF NOT EXISTS] table_name
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
    ...
)
```
说明：
1. 临时表只支持Memory表引擎
2. 临时表不属于任何数据库，创建临时表的定义没有数据库参数和标引擎参数。
3. 临时表和常规表的表名称相同则优先读取临时表的数据。

```bash
Clickhouse> create table t_temp(desc varchar(254)) engine=Memory;

CREATE TABLE t_temp
(
    `desc` varchar(254)
)
ENGINE = Memory

Ok.

0 rows in set. Elapsed: 0.002 sec. 
```
```bash
Clickhouse> insert into t_temp values('clickhouse');

INSERT INTO t_temp VALUES

Ok.

1 rows in set. Elapsed: 0.001 sec. 
```
```bash
Clickhouse> create temporary table t_temp(createtime datetime);

CREATE TEMPORARY TABLE t_temp
(
    `createtime` datetime
)

Ok.

0 rows in set. Elapsed: 0.001 sec. 
```
```bash
Clickhouse> insert into t_temp values(now());

INSERT INTO t_temp VALUES

Ok.

1 rows in set. Elapsed: 0.005 sec. 
```

```bash
Clickhouse> select * from t_temp;

SELECT *
FROM t_temp

┌──────────createtime─┐
│ 2020-07-13 23:45:51 │
└─────────────────────┘

1 rows in set. Elapsed: 0.001 sec. 
```

### 注意:
临时表平常不怎么使用，更多的应用于clickhouse的内部在集群之间传播的。

### 参考
https://clickhouse.tech/docs/en/sql-reference/statements/alter/
https://clickhouse.tech/docs/en/sql-reference/statements/create/table/