# RDD 编程学习

## RDD创建

- 从本地文件系统加载数据

  ```scala
  scala> val lines = sc.textFile("file:///usr/local/spark/mycode/rdd/word.txt"
  ```

- 从分布式文件系统HDFS中加载数据（下面三行语句是等价的）

  ```scala
  scala> val lines = sc.textFile("hdfs://localhost:9000/user/hadoop/word.txt")
  scala> val lines = sc.textFile("/user/hadoop/word.txt")
  scala> val lines = sc.textFile("word.txt")
  ```

- 通过并行集合（数组）创建RDD

  ```scala
  scala> val array = Array(1,2,3,4,5)
  array: Array[Int] = Array(1, 2, 3, 4, 5)
  
  scala> val rdd = sc.parallelize(array)
  rdd: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[2] at parallelize at <console>:26
  ```

  ```scala
  scala> val list = List(1,2,3,4,5)
  list: List[Int] = List(1, 2, 3, 4, 5)
  
  scala> val rdd2 = sc.parallelize(list)
  rdd2: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[3] at parallelize at <console>:26
  ```

## RDD 持久化

  ```scala
  // 表示将RDD作为反序列化的对象存储于JVM中，如果内存不足，就要按照LRU原则替换内容
  persist(MEMORY_ONLY) 
  // 表示将RDD作为反序列化的对象存储于JVM中，如果内存不足，超出的分区将会被存放在硬盘上
  persist(MEMORY_AND_DISK)
  rdd.cache() //等价 persist(MEMORY_ONLY) 
  // 手动地把持久化的RDD从缓存中移除
  unpersist()
  ```

## 分区

- 分区原则：分区的个数尽量等于集群中CPU核心(core)数目

- spark 默认分区数目配置：spark.default.parallelism

- 本地模式：默认为本地机器的CPU数目，若设置了local[N],则默认为N

- mesos: 默认分区为8

- standalone或yarn:  在集群中所有CPU核心数目总和 和 2，两者取较大值为默认值

- parallelize: sc.parallelize(array, 2) 设置2个分区， 如果没有指定分区，则默认spark.default.parallelism

- textFile:  如果没有指定分区，则默认min(spark.default.parallelism, 2)

- 如果从 HDFS 读取文件，则分区数为文件分片数（比如，128MB/片）

- 手动设置分区： 

  - 创建RDD时指定：sc.textFile(path, partitionnum)

  - 转换RDD时，直接调用 repartition方法即可

    ```scala
    val rdd2 = data.repartition(1)
    rdd2.partitions.size  //1
    var rdd2 = data.data.repartition(4)
    rdd2.partitions.size  //4
    ```

- 自定义分区
```scala
import org.apache.spark.{Partitioner, SparkConf, SparkContext}

//自定义分区数，需继承 Partitioner 类
class UsridPartitioner(numParts: Int) extends Partitioner{
  //覆盖分区数
  override def numPartitions: Int = numParts
  //覆盖分区号获取函数
  override def getPartition(key: Any): Int = {
    key.toString.toInt%10
  }
}

object Test {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf();
    val sc = new SparkContext(conf);
    //模拟 5个分区的数据
    val data = sc.parallelize(1 to 10, 5);
    //根据尾号转变为10个分区，分写到 10 个文件
    //data.map((_,1)) 等价于 data.map(x=>(x,1))
    data.map((_,1)).partitionBy(new
        UsridPartitioner(10)).map(_._1).saveAsTextFile("file:///usr/local/output");
  }
}
```

- 打印：rdd.foreach(println) 或者 rdd.map(println)
- 集群中打印 rdd.collect().foreach(println) 或者  rdd.take(100).foreach(println)

## Pair RDD 

### 创建

-  从文件中加载

- 从array list 创建

### 方法

- reduceByKey

- groupByKey

  ```scala
  val words = Array("one", "two","two","three","three")
  val wordPairsRDD = sc.parallelize(words).map(word=>(word,1))
  val wordCountsWithReduce = wordPairsRDD.reduceByKey(_+_)
  val wordCountsWithGroup = wordPairsRDD.groupByKey().map(t=>(t._1,t._2.sum))
  ```

  wordCountsWithReduce 和 wordCountsWithGroup 是完全一样的，但是他们的内部运算过程是不同的