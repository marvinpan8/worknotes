# Spark强大的函数扩展功能

https://www.cnblogs.com/tomato0906/articles/7082326.html  2017-06-26

在数据分析领域中，没有人能预见所有的数据运算，以至于将它们都内置好，一切准备完好，用户只需要考虑用，万事大吉。扩展性是一个平台的生存之本，一个封闭的平台如何能够拥抱变化？在对数据进行分析时，无论是算法也好，分析逻辑也罢，最好的重用单位自然还是： **函数** 。

故而，对于一个大数据处理平台而言，倘若不能支持函数的扩展，确乎是不可想象的。Spark首先是一个开源框架，当我们发现一些函数具有通用的性质，自然可以考虑contribute给社区，直接加入到Spark的源代码中。我们欣喜地看到随着Spark版本的演化，确实涌现了越来越多对于数据分析师而言称得上是一柄柄利器的强大函数，例如博客文章《 [Spark 1.5 DataFrame API Highlights: Date/Time/String Handling, Time Intervals, and UDAFs](https://databricks.com/blog/2015/09/16/spark-1-5-dataframe-api-highlights-datetimestring-handling-time-intervals-and-udafs.html) 》介绍了在1.5中为DataFrame提供了丰富的处理日期、时间和字符串的函数；以及在Spark SQL 1.4中就引入的 [Window Function](https://databricks.com/blog/2015/07/15/introducing-window-functions-in-spark-sql.html) 。

然而，针对特定领域进行数据分析的函数扩展，Spark提供了更好地置放之处，那就是所谓的“UDF（User Defined Function）”。

UDF的引入极大地丰富了Spark SQL的表现力。一方面，它让我们享受了利用Scala（当然，也包括Java或Python）更为自然地编写代码实现函数的福利，另一方面，又能精简SQL（或者DataFrame的API），更加写意自如地完成复杂的数据分析。尤其采用SQL语句去执行数据分析时，UDF帮助我们在SQL函数与Scala函数之间左右逢源，还可以在一定程度上化解不同数据源具有歧异函数的尴尬。想想不同关系数据库处理日期或时间的函数名称吧！

用Scala编写的UDF与普通的Scala函数没有任何区别，唯一需要多执行的一个步骤是要让SQLContext注册它。例如：

```scala
def len(bookTitle: String):Int = bookTitle.length

sqlContext.udf.register("len", len _)

val booksWithLongTitle = sqlContext.sql("select title, author from books where len(title) > 10")
```

编写的UDF可以放到SQL语句的fields部分，也可以作为where、groupBy或者having子句的一部分。

既然是UDF，它也得保持足够的特殊性，否则就完全与Scala函数泯然众人也。这一特殊性不在于函数的实现，而是思考函数的角度，需要将UDF的参数视为数据表的某个列。例如上面 `len` 函数的参数 `bookTitle` ，虽然是一个普通的字符串，但当其代入到Spark SQL的语句中，实参 `title` 实际上是表中的一个列（可以是列的别名）。

当然，我们也可以在使用UDF时，传入常量而非表的列名。让我们稍稍修改一下刚才的函数，让长度10作为函数的参数传入：

```scala
def lengthLongerThan(bookTitle: String, length: Int): Boolean = bookTitle.length > length

sqlContext.udf.register("longLength", lengthLongerThan _)

val booksWithLongTitle = sqlContext.sql("select title, author from books where longLength(title, 10)")
```

若使用DataFrame的API，则可以以字符串的形式将UDF传入：

```scala
val booksWithLongTitle = dataFrame.filter("longLength(title, 10)")
```

**DataFrame的API也可以接收Column对象**，可以用 `$` 符号来包裹一个字符串表示一个Column。 `$` 是定义在SQLContext对象implicits中的一个隐式转换。此时，UDF的定义也不相同，不能直接定义Scala函数，而是要用定义在 `org.apache.spark.sql.functions` 中的udf方法来接收一个函数。这种方式无需register：

```scala
import org.apache.spark.sql.functions._

val longLength = udf((bookTitle: String, length: Int) => bookTitle.length > length)

import sqlContext.implicits._
val booksWithLongTitle = dataFrame.filter(longLength($"title", $"10"))
```

注意，代码片段中的 `sqlContext` 是之前已经实例化的SQLContext对象。

不幸，运行这段代码会抛出异常：

```properties
cannot resolve '10' given input columns id, title, author, price, publishedDate;
```

因为采用 `$` 来包裹一个常量，会让Spark错以为这是一个Column。这时，需要定义在org.apache.spark.sql.functions 中的 `lit` 函数来帮助：

```scala
val booksWithLongTitle = dataFrame.filter(longLength($"title", lit(10)))
```

**普通的UDF却也存在一个缺陷，就是无法在函数内部支持对表数据的聚合运算。**例如，当我要对销量执行年度同比计算，就需要对当年和上一年的销量分别求和，然后再利用同比公式进行计算。此时，UDF就无能为力了。

该UDAF（User Defined Aggregate Function）粉墨登场的时候了。

Spark为所有的UDAF定义了一个父类 `UserDefinedAggregateFunction` 。要继承这个类，需要实现父类的几个抽象方法：

```scala
def inputSchema: StructType

def bufferSchema: StructType

def dataType: DataType

def deterministic: Boolean

def initialize(buffer: MutableAggregationBuffer): Unit

def update(buffer: MutableAggregationBuffer, input: Row): Unit

def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit

def evaluate(buffer: Row): Any
```

可以将 `inputSchema` 理解为UDAF与DataFrame列有关的输入样式。例如年同比函数需要对某个可以运算的指标与时间维度进行处理，就需要在 `inputSchema` 中定义它们。

```scala
def inputSchema: StructType = {
    StructType(StructField("metric", DoubleType) :: StructField("timeCategory", DateType) :: Nil)
  }
```

代码创建了拥有两个 `StructField` 的 `StructType` 。 `StructField` 的名字并没有特别要求，完全可以认为是两个内部结构的列名占位符。至于UDAF具体要操作DataFrame的哪个列，取决于调用者，但前提是数据类型必须符合事先的设置，如这里的 `DoubleType` 与 `DateType` 类型。这两个类型被定义在 `org.apache.spark.sql.types` 中。

`bufferSchema` 用于定义存储聚合运算时产生的中间数据结果的Schema，例如我们需要存储当年与上一年的销量总和，就需要定义两个 `StructField` ：

```scala
def bufferSchema: StructType = {
    StructType(StructField("sumOfCurrent", DoubleType) :: StructField("sumOfPrevious", DoubleType) :: Nil)
  }
```

`dataType` 标明了UDAF函数的返回值类型， `deterministic` 是一个布尔值，用以标记针对给定的一组输入，UDAF是否总是生成相同的结果。

顾名思义， `initialize` 就是对聚合运算中间结果的初始化，在我们这个例子中，两个求和的中间值都被初始化为0d：

```scala
def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer.update(0, 0.0)
    buffer.update(1, 0.0)
  }
```

`update` 函数的第一个参数为bufferSchema中两个Field的索引，默认以0开始，所以第一行就是针对 “sumOfCurrent” 的求和值进行初始化。

UDAF的核心计算都发生在 `update` 函数中。在我们这个例子中，需要用户设置计算同比的时间周期。这个时间周期值属于外部输入，但却并非 `inputSchema` 的一部分，所以应该从UDAF对应类的构造函数中传入。我为时间周期定义了一个样例类，且对于同比函数，我们只要求输入当年的时间周期，上一年的时间周期可以通过对年份减1来完成：

```scala
case class DateRange(startDate: Timestamp, endDate: Timestamp) {
  def in(targetDate: Date): Boolean = {
    targetDate.before(endDate) && targetDate.after(startDate)
  }
}

class YearOnYearBasis(current: DateRange) extends UserDefinedAggregateFunction {
  def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    if (current.in(input.getAs[Date](1))) {
      buffer(0) = buffer.getAs[Double](0) + input.getAs[Double](0)
    }
    val previous = DateRange(subtractOneYear(current.startDate), subtractOneYear(current.endDate))
    if (previous.in(input.getAs[Date](1))) {
      buffer(1) = buffer.getAs[Double](0) + input.getAs[Double](0)
    }
  }
}
```

`update` 函数的第二个参数 `input: Row` 对应的并非DataFrame的行，而是被inputSchema投影了的行。以本例而言，每一个input就应该只有两个Field的值。倘若我们在调用这个UDAF函数时，分别传入了 **销量** 和 **销售日期** 两个列的话，则 `input(0)` 代表的就是销量， `input(1)` 代表的就是销售日期。

`merge` 函数负责合并两个聚合运算的buffer，再将其存储到 `MutableAggregationBuffer` 中：

```scala
def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    buffer1(0) = buffer1.getAs[Double](0) + buffer2.getAs[Double](0)
    buffer1(1) = buffer1.getAs[Double](1) + buffer2.getAs[Double](1)
  }
```

最后，由 `evaluate` 函数完成对聚合Buffer值的运算，得到最后的结果：

```scala
def evaluate(buffer: Row): Any = {
    if (buffer.getDouble(1) == 0.0)
      0.0
    else
      (buffer.getDouble(0) - buffer.getDouble(1)) / buffer.getDouble(1) * 100
  }
```

假设我们创建了这样一个简单的DataFrame：

```scala
val conf = new SparkConf().setAppName("TestUDF").setMaster("local[*]")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)

    import sqlContext.implicits._

    val sales = Seq(
      (1, "Widget Co", 1000.00, 0.00, "AZ", "2014-01-01"),
      (2, "Acme Widgets", 2000.00, 500.00, "CA", "2014-02-01"),
      (3, "Widgetry", 1000.00, 200.00, "CA", "2015-01-11"),
      (4, "Widgets R Us", 2000.00, 0.0, "CA", "2015-02-19"),
      (5, "Ye Olde Widgete", 3000.00, 0.0, "MA", "2015-02-28")
    )

    val salesRows = sc.parallelize(sales, 4)
    val salesDF = salesRows.toDF("id", "name", "sales", "discount", "state", "saleDate")
    salesDF.registerTempTable("sales")
```

那么，要使用之前定义的UDAF，则需要实例化该UDAF类，然后再通过udf进行注册：

```scala
val current = DateRange(Timestamp.valueOf("2015-01-01 00:00:00"), Timestamp.valueOf("2015-12-31 00:00:00"))
    val yearOnYear = new YearOnYearBasis(current)

    sqlContext.udf.register("yearOnYear", yearOnYear)
    val dataFrame = sqlContext.sql("select yearOnYear(sales, saleDate) as yearOnYear from sales")
    dataFrame.show()
```

在使用上，除了需要对UDAF进行实例化之外，与普通的UDF使用没有任何区别。但显然，UDAF更加地强大和灵活。如果Spark自身没有提供符合你需求的函数，且需要进行较为复杂的聚合运算，UDAF是一个不错的选择。

通过Spark提供的UDF与UDAF，你可以慢慢实现属于自己行业的函数库，让Spark SQL变得越来越强大，对于使用者而言，却能变得越来越简单。