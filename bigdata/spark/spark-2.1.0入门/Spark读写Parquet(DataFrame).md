# Spark入门：读写Parquet(DataFrame)

 林子雨老师 2016年11月19日 http://dblab.xmu.edu.cn/blog/1091-2/

---

Spark SQL可以支持Parquet、JSON、Hive等数据源，并且可以通过JDBC连接外部数据源。前面的介绍中，我们已经涉及到了JSON、文本格式的加载，这里不再赘述。这里介绍Parquet，下一节会介绍JDBC数据库连接。

Parquet是一种流行的列式存储格式，可以高效地存储具有嵌套字段的记录。Parquet是语言无关的，而且不与任何一种数据处理框架绑定在一起，适配多种语言和组件，能够与Parquet配合的组件有：

- 查询引擎: Hive, Impala, Pig, Presto, Drill, Tajo, HAWQ, IBM Big SQL
- 计算框架: MapReduce, **Spark**, Cascading, Crunch, Scalding, Kite
- 数据模型: Avro, Thrift, Protocol Buffers, POJOs

Spark已经为我们提供了parquet样例数据，就保存在“/usr/local/spark/examples/src/main/resources/”这个目录下，有个users.parquet文件，这个文件格式比较特殊，如果你用vim编辑器打开，或者用cat命令查看文件内容，肉眼是一堆乱七八糟的东西，是无法理解的。只有被加载到程序中以后，Spark会对这种格式进行解析，然后我们才能理解其中的数据。
下面代码演示了如何从parquet文件中加载数据生成DataFrame。

```scala
scala> import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.SQLContext
 
scala> val sqlContext = new SQLContext(sc)
sqlContext: org.apache.spark.sql.SQLContext = org.apache.spark.sql.SQLContext@35f18100
 
scala> val users = sqlContext.read.load("file:///usr/local/spark/examples/src/main/resources/users.parquet")
users: org.apache.spark.sql.DataFrame = [name: string, favorite_color: string, favorite_numbers: array<int>]
 
scala> users.registerTempTable("usersTempTab")
 
scala> val usersRDD =sqlContext.sql("select * from usersTempTab").rdd
usersRDD: org.apache.spark.rdd.RDD[org.apache.spark.sql.Row] = MapPartitionsRDD[13] at rdd at <console>:34
 
scala> usersRDD.foreach(t=>println("name:"+t(0)+"  favorite color:"+t(1)))
name:Alyssa  favorite color:null
name:Ben  favorite color:red
```

下面介绍如何将DataFrame保存成parquet文件。

进入spark-shell执行下面命令：

```scala
scala> import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.SQLContext
 
scala> val sqlContext = new SQLContext(sc)
sqlContext: org.apache.spark.sql.SQLContext = org.apache.spark.sql.SQLContext@4a65c40
 
scala> val df = sqlContext.read.json("file:///usr/local/spark/examples/src/main/resources/people.json")
df: org.apache.spark.sql.DataFrame = [age: bigint, name: string]
 
scala> df.select("name","age").write.format("parquet").save("file:////usr/local/spark/examples/src/main/resources/newpeople.parquet")
```

上述过程执行结束后，可以打开第二个终端窗口，在Shell命令提示符下查看新生成的newpeople.parquet：

```bash
$ cd  /usr/local/spark/examples/src/main/resources/
$ ls
```

上面命令执行后，可以看到”/usr/local/spark/examples/src/main/resources/”这个目录下多了一个newpeople.parquet，不过，注意，这不是一个文件，而是一个目录（不要被newpeople.parquet中的圆点所迷惑，文件夹名称也可以包含圆点），也就是说，df.select(“name”,”age”).write.format(“parquet”).save（）括号里面的**参数是文件夹，不是文件名**。下面我们可以进入newpeople.parquet目录，会发现下面4个文件：

```properties
_common_metadata  
_metadata  
part-r-00000-ad565c11-d91b-4de7-865b-ea17f8e91247.gz.parquet  
_SUCCESS
```

这4个文件都是刚才保存生成的。现在问题来了，如果我们要再次把这个刚生成的数据又加载到DataFrame中，应该加载哪个文件呢？很简单，只要加载newpeople.parquet目录即可，而不是加载这4个文件，语句如下：

```scala
val users = sqlContext.read.load("file:///usr/local/spark/examples/src/main/resources/newpeople.parquet")
```