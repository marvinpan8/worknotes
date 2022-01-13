# Spark2.1.0入门：从RDD转换得到DataFrame

林子雨老师 2017年2月27日 (updated: 2017年3月20日) http://dblab.xmu.edu.cn/blog/1360-2/

---

[[返回Spark教程首页](http://dblab.xmu.edu.cn/blog/spark/)]
Spark官网提供了两种方法来实现从RDD转换得到DataFrame，

第一种方法是，利用反射来推断包含特定类型对象的RDD的schema，适用对已知数据结构的RDD转换；

第二种方法是，使用编程接口，构造一个schema并将其应用在已知的RDD上。

### 利用反射机制推断RDD模式

在利用反射机制推断RDD模式时，需要首先定义一个case class，因为，只有case class才能被Spark隐式地转换为DataFrame。
下面是在spark-shell中执行命令以及反馈的信息：

```scala
scala> import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder
import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder
 
scala> import org.apache.spark.sql.Encoder
import org.apache.spark.sql.Encoder
 
scala> import spark.implicits._  //导入包，支持把一个RDD隐式转换为一个DataFrame
import spark.implicits._
 
scala> case class Person(name: String, age: Long)  //定义一个case class
defined class Person
 
scala> val peopleDF = spark.sparkContext.textFile("file:///usr/local/spark/examples/src/main/resources/people.txt").map(_.split(",")).map(attributes => Person(attributes(0), attributes(1).trim.toInt)).toDF()
peopleDF: org.apache.spark.sql.DataFrame = [name: string, age: bigint]
 
scala> peopleDF.createOrReplaceTempView("people")  //必须注册为临时表才能供下面的查询使用
 
scala> val personsRDD = spark.sql("select name,age from people where age > 20")
//最终生成一个DataFrame
personsRDD: org.apache.spark.sql.DataFrame = [name: string, age: bigint]
scala> personsRDD.map(t => "Name:"+t(0)+","+"Age:"+t(1)).show()  //DataFrame中的每个元素都是一行记录，包含name和age两个字段，分别用t(0)和t(1)来获取值
 
+------------------+
|             value|
+------------------+
|Name:Michael,Age:29|
|   Name:Andy,Age:30|
+------------------+
```

### 使用编程方式定义RDD模式

当无法提前定义case class时，就需要采用编程方式定义RDD模式。

```scala
scala> import org.apache.spark.sql.types._
import org.apache.spark.sql.types._
 
scala> import org.apache.spark.sql.Row
import org.apache.spark.sql.Row
 
//生成 RDD
scala> val peopleRDD = spark.sparkContext.textFile("file:///usr/local/spark/examples/src/main/resources/people.txt")
peopleRDD: org.apache.spark.rdd.RDD[String] = file:///usr/local/spark/examples/src/main/resources/people.txt MapPartitionsRDD[1] at textFile at <console>:26
 
//定义一个模式字符串
scala> val schemaString = "name age"
schemaString: String = name age
 
//根据模式字符串生成模式
scala> val fields = schemaString.split(" ").map(fieldName => StructField(fieldName, StringType, nullable = true))
fields: Array[org.apache.spark.sql.types.StructField] = Array(StructField(name,StringType,true), StructField(age,StringType,true))
 
scala> val schema = StructType(fields)
schema: org.apache.spark.sql.types.StructType = StructType(StructField(name,StringType,true), StructField(age,StringType,true))
//从上面信息可以看出，schema描述了模式信息，模式中包含name和age两个字段
 
//对peopleRDD 这个RDD中的每一行元素都进行解析val peopleDF = spark.read.format("json").load("examples/src/main/resources/people.json")
 
 
scala> val rowRDD = peopleRDD.map(_.split(",")).map(attributes => Row(attributes(0), attributes(1).trim))
rowRDD: org.apache.spark.rdd.RDD[org.apache.spark.sql.Row] = MapPartitionsRDD[3] at map at <console>:29
 
scala> val peopleDF = spark.createDataFrame(rowRDD, schema)
peopleDF: org.apache.spark.sql.DataFrame = [name: string, age: string]
 
//必须注册为临时表才能供下面查询使用
scala> peopleDF.createOrReplaceTempView("people")
 
scala> val results = spark.sql("SELECT name,age FROM people")
results: org.apache.spark.sql.DataFrame = [name: string, age: string]
 
scala> results.map(attributes => "name: " + attributes(0)+","+"age:"+attributes(1)).show()
+--------------------+
|               value|
+--------------------+
|name: Michael,age:29|
|   name: Andy,age:30|
| name: Justin,age:19|
+--------------------+
```

在上面的代码中，people.map(_.split(“,”))实际上和people.map(line => line.split(“,”))这种表述是等价的，作用是对people这个RDD中的每一行元素都进行解析。比如，people这个RDD的第一行是：

```
Michael, 29
```

这行内容经过people.map(_.split(“,”))操作后，就得到一个集合{Michael,29}。后面经过map(p => Row(p(0), p(1).trim))操作时，这时的p就是这个集合{Michael,29}，这时p(0)就是Micheael，p(1)就是29，map(p => Row(p(0), p(1).trim))就会生成一个Row对象，这个对象里面包含了两个字段的值，这个Row对象就构成了rowRDD中的其中一个元素。因为people有3行文本，所以，最终，rowRDD中会包含3个元素，每个元素都是org.apache.spark.sql.Row类型。实际上，Row对象只是对基本数据类型（比如整型或字符串）的数组的封装，本质就是一个定长的字段数组。
peopleDF = spark.createDataFrame(rowRDD, schema)，这条语句就相当于建立了rowRDD数据集和模式之间的对应关系，从而我们就知道对于rowRDD的每行记录，第一个字段的名称是schema中的“name”，第二个字段的名称是schema中的“age”。

### 把RDD保存成文件

这里介绍如何把RDD保存成文本文件，后面还会介绍其他格式的保存。

#### 第1种保存方法

进入spark-shell执行下面命令：

```scala
scala> val peopleDF = spark.read.format("json").load("file:///usr/local/spark/examples/src/main/resources/people.json")
peopleDF: org.apache.spark.sql.DataFrame = [age: bigint, name: string]
 
scala> peopleDF.select("name", "age").write.format("csv").save("file:///usr/local/spark/mycode/newpeople.csv")
```

可以看出，这里使用select(“name”, “age”)确定要把哪些列进行保存，然后调用write.format(“csv”).save ()保存成csv文件。在后面小节中，我们还会介绍其他保存方式。
另外，write.format()支持输出 json,parquet, jdbc, orc, libsvm, csv, text等格式文件，如果要输出文本文件，可以采用write.format(“text”)，但是，需要注意，只有select()中只存在一个列时，才允许保存成文本文件，如果存在两个列，比如select(“name”, “age”)，就不能保存成文本文件。

上述过程执行结束后，可以打开第二个终端窗口，在Shell命令提示符下查看新生成的newpeople.csv：

```bash
cd  /usr/local/spark/mycode/
ls
```

可以看到/usr/local/spark/mycode/这个目录下面有个newpeople.csv文件夹（注意，不是文件），这个文件夹中包含下面两个文件：

```
part-r-00000-33184449-cb15-454c-a30f-9bb43faccac1.csv 
_SUCCESS
```

不用理会_SUCCESS这个文件，只要看一下part-r-00000-33184449-cb15-454c-a30f-9bb43faccac1.csv这个文件，可以用vim编辑器打开这个文件查看它的内容，该文件内容如下：

```
Michael,
Andy,30
Justin,19
```

因为people.json文件中，Michael这个名字不存在对应的age，所以，上面第一行逗号后面没有内容。
如果我们要再次把newpeople.csv中的数据加载到RDD中，可以直接使用newpeople.csv目录名称，而不需要使用part-r-00000-33184449-cb15-454c-a30f-9bb43faccac1.csv 文件，如下：

```scala
scala> val textFile = sc.textFile("file:///usr/local/spark/mycode/newpeople.csv")
textFile: org.apache.spark.rdd.RDD[String] = file:///usr/local/spark/mycode/newpeople.csv MapPartitionsRDD[1] at textFile at <console>:24
scala> textFile.foreach(println)
Justin,19
Michael,
Andy,30
```

#### 第2种保存方法

进入spark-shell执行下面命令：

```scala
scala> val peopleDF = spark.read.format("json").load("file:///usr/local/spark/examples/src/main/resources/people.json")
peopleDF: org.apache.spark.sql.DataFrame = [age: bigint, name: string]
 
scala> df.rdd.saveAsTextFile("file:///usr/local/spark/mycode/newpeople.txt")
```

可以看出，我们是把DataFrame转换成RDD，然后调用saveAsTextFile()保存成文本文件。在后面小节中，我们还会介绍其他保存方式。
上述过程执行结束后，可以打开第二个终端窗口，在Shell命令提示符下查看新生成的newpeople.txt：

```bash
$ cd  /usr/local/spark/mycode/
$ ls
```

可以看到/usr/local/spark/mycode/这个目录下面有个newpeople.txt文件夹（注意，不是文件），这个文件夹中包含下面两个文件：

```
part-00000  
_SUCCESS
```

不用理会_SUCCESS这个文件，只要看一下part-00000这个文件，可以用vim编辑器打开这个文件查看它的内容，该文件内容如下：

```
[null,Michael]
[30,Andy]
[19,Justin]
```

如果我们要再次把newpeople.txt中的数据加载到RDD中，可以直接使用newpeople.txt目录名称，而不需要使用part-00000文件，如下：

```scala
scala> val textFile = sc.textFile("file:///usr/local/spark/mycode/newpeople.txt")
textFile: org.apache.spark.rdd.RDD[String] = file:///usr/local/spark/mycode/newpeople.txt MapPartitionsRDD[11] at textFile at <console>:28
 
scala> textFile.foreach(println)
[null,Michael]
[30,Andy]
[19,Justin]
```