# Spark2.1.0入门：DataFrame的创建

林子雨老师 2017年2月27日 (updated: 2017年3月19日) http://dblab.xmu.edu.cn/blog/1359-2/

---

[[返回Spark教程首页](http://dblab.xmu.edu.cn/blog/spark/)]

从Spark2.0以上版本开始，Spark使用全新的SparkSession接口替代Spark1.6中的SQLContext及HiveContext接口来实现其对数据加载、转换、处理等功能。SparkSession实现了SQLContext及HiveContext所有功能。

SparkSession支持从不同的数据源加载数据，并把数据转换成DataFrame，并且支持把DataFrame转换成SQLContext自身中的表，然后使用SQL语句来操作数据。SparkSession亦提供了HiveQL以及其他依赖于Hive的功能的支持。

下面我们就介绍如何使用SparkSession来创建DataFrame。
请进入Linux系统，打开“终端”，进入Shell命令提示符状态。

首先，请找到样例数据。 Spark已经为我们提供了几个样例数据，就保存在“/usr/local/spark/examples/src/main/resources/”这个目录下，这个目录下有两个样例数据people.json和people.txt。
people.json文件的内容如下：

```
{"name":"Michael"}
{"name":"Andy", "age":30}
{"name":"Justin", "age":19}
```

people.txt文件的内容如下：

```
Michael, 29
Andy, 30
Justin, 19
```

下面我们就介绍如何从people.json文件中读取数据并生成DataFrame并显示数据（从people.txt文件生成DataFrame需要后面将要介绍的另外一种方式）。
请使用如下命令打开spark-shell：

```bash
$ cd /usr/local/spark
$ ./bin/spark-shell
```

进入到spark-shell状态后执行下面命令：

```scala
scala> import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.SparkSession
 
scala> val spark=SparkSession.builder().getOrCreate()
spark: org.apache.spark.sql.SparkSession = org.apache.spark.sql.SparkSession@2bdab835
 
//使支持RDDs转换为DataFrames及后续sql操作
scala> import spark.implicits._
import spark.implicits._
 
scala> val df = spark.read.json("file:///usr/local/spark/examples/src/main/resources/people.json")
df: org.apache.spark.sql.DataFrame = [age: bigint, name: string]
 
scala> df.show()
+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+
```

现在，我们可以执行一些常用的DataFrame操作。

```scala
// 打印模式信息
scala> df.printSchema()
root
 |-- age: long (nullable = true)
 |-- name: string (nullable = true)
 
// 选择多列
scala> df.select(df("name"),df("age")+1).show()
+-------+---------+
|   name|(age + 1)|
+-------+---------+
|Michael|     null|
|   Andy|       31|
| Justin|       20|
+-------+---------+
 
// 条件过滤
scala> df.filter(df("age") > 20 ).show()
+---+----+
|age|name|
+---+----+
| 30|Andy|
+---+----+
 
// 分组聚合
scala> df.groupBy("age").count().show()
+----+-----+
| age|count|
+----+-----+
|  19|    1|
|null|    1|
|  30|    1|
+----+-----+
 
// 排序
scala> df.sort(df("age").desc).show()
+----+-------+
| age|   name|
+----+-------+
|  30|   Andy|
|  19| Justin|
|null|Michael|
+----+-------+
 
//多列排序
scala> df.sort(df("age").desc, df("name").asc).show()
+----+-------+
| age|   name|
+----+-------+
|  30|   Andy|
|  19| Justin|
|null|Michael|
+----+-------+
 
//对列进行重命名
scala> df.select(df("name").as("username"),df("age")).show()
+--------+----+
|username| age|
+--------+----+
| Michael|null|
|    Andy|  30|
|  Justin|  19|
+--------+----+
```