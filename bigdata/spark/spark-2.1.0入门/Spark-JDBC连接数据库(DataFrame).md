# Spark2.1.0入门：通过JDBC连接数据库(DataFrame)

林子雨老师 2017年2月27日 (updated: 2017年3月20日) http://dblab.xmu.edu.cn/blog/1366-2/

---

[[返回Spark教程首页](http://dblab.xmu.edu.cn/blog/spark/)]

这里以关系数据库MySQL为例。首先，请参考厦门大学数据库实验室博客教程（[Ubuntu安装MySQL](http://dblab.xmu.edu.cn/blog/install-mysql/)），在Linux系统中安装好MySQL数据库。这里假设你已经成功安装了MySQL数据库。下面我们要新建一个测试Spark程序的数据库，数据库名称是“spark”，表的名称是“student”。

请执行下面命令在Linux中启动MySQL数据库，并完成数据库和表的创建，以及样例数据的录入：

```bash
$ service mysql start
$ mysql -u root -p
//屏幕会提示你输入密码
```

输入密码后，你就可以进入“mysql>”命令提示符状态，然后就可以输入下面的SQL语句完成数据库和表的创建：

```mysql
mysql> create database spark;
mysql> use spark;
mysql> create table student (id int(4), name char(20), gender char(4), age int(4));
mysql> insert into student values(1,'Xueqian','F',23);
mysql> insert into student values(2,'Weiliang','M',24);
mysql> select * from student;
```

上面已经创建好了我们所需要的MySQL数据库和表，下面我们编写Spark应用程序连接MySQL数据库并且读写数据。

Spark支持通过JDBC方式连接到其他数据库获取数据生成DataFrame。

首先，请进入Linux系统（本教程统一使用hadoop用户名登录），打开火狐（FireFox）浏览器，下载一个MySQL的JDBC驱动（[下载](http://dev.mysql.com/downloads/connector/j/)）。在火狐浏览器中下载时，一般默认保存在hadoop用户的当前工作目录的“下载”目录下，所以，可以打开一个终端界面，输入下面命令查看：

```bash
$ cd ~
$ cd 下载
```

就可以看到刚才下载到的MySQL的JDBC驱动程序，文件名称为mysql-connector-java-5.1.40.tar.gz（你下载的版本可能和这个不同）。现在，使用下面命令，把该驱动程序拷贝到spark的安装目录下：

```bash
$ sudo tar -zxf ~/下载/mysql-connector-java-5.1.40.tar.gz -C /usr/local/spark/jars
$ cd /usr/local/spark/jars
$ ls
```

这时就可以在/usr/local/spark/jars目录下看到这个驱动程序文件所在的文件夹mysql-connector-java-5.1.40，进入这个文件夹，就可以看到驱动程序文件mysql-connector-java-5.1.40-bin.jar。
请输入下面命令启动已经安装在Linux系统中的mysql数据库（如果前面已经启动了MySQL数据库，这里就不用重复启动了）。

```bash
$ service mysql start
```

下面，我们要启动一个spark-shell，而且启动的时候，要附加一些参数。启动Spark Shell时，必须指定mysql连接驱动jar包。

```bash
$ cd /usr/local/spark
$ ./bin/spark-shell \
--jars /usr/local/spark/jars/mysql-connector-java-5.1.40/mysql-connector-java-5.1.40-bin.jar \
--driver-class-path /usr/local/spark/jars/mysql-connector-java-5.1.40/mysql-connector-java-5.1.40-bin.jar
```

上面的命令行中，在一行的末尾加入斜杠\，是为了告诉spark-shell，命令还没有结束。

启动进入spark-shell以后，可以执行以下命令连接数据库，读取数据，并显示：

```scala
scala> val jdbcDF = spark.read.format("jdbc").option("url", "jdbc:mysql://localhost:3306/spark").option("driver","com.mysql.jdbc.Driver").option("dbtable", "student").option("user", "root").option("password", "hadoop").load()
Fri Dec 02 11:56:56 CST 2016 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
jdbcDF: org.apache.spark.sql.DataFrame = [id: int, name: string ... 2 more fields]
 
scala> jdbcDF.show()
Fri Dec 02 11:57:30 CST 2016 WARN: Establishing SSL connection without server's identity verification is not recommended. According to MySQL 5.5.45+, 5.6.26+ and 5.7.6+ requirements SSL connection must be established by default if explicit option isn't set. For compliance with existing applications not using SSL the verifyServerCertificate property is set to 'false'. You need either to explicitly disable SSL by setting useSSL=false, or set useSSL=true and provide truststore for server certificate verification.
+---+--------+------+---+
| id|    name|gender|age|
+---+--------+------+---+
|  1| Xueqian|     F| 23|
|  2|Weiliang|     M| 24|
+---+--------+------+---+
```

下面我们再来看一下如何往MySQL中写入数据。
为了看到MySQL数据库在Spark程序执行前后发生的变化，我们先在Linux系统中新建一个终端，使用下面命令查看一下MySQL数据库中的数据库spark中的表student的内容：

```bash
$ service mysql start //如果前面已经启动MySQL数据库，这里不用再执行这条命令
$ mysql -u root -p
```

执行上述命令后，屏幕上会提示你输入MySQL数据库密码，我们这里给数据库设置的用户是root，密码是hadoop。
因为之前我们已经在MySQL数据库中创建了一个名称为spark的数据库，并创建了一个名称为student的表，现在查看一下：

```mysql
mysql>  use spark;
Database changed
 
mysql> select * from student;
//上面命令执行后返回下面结果
+------+----------+--------+------+
| id   | name     | gender | age  |
+------+----------+--------+------+
|    1 | Xueqian  | F      |   23 |
|    2 | Weiliang | M      |   24 |
+------+----------+--------+------+
2 rows in set (0.00 sec)
 
```

现在我们开始在spark-shell中编写程序，往spark.student表中插入两条记录。
下面，我们要启动一个spark-shell，而且启动的时候，要附加一些参数。启动Spark Shell时，必须指定mysql连接驱动jar包（如果你前面已经采用下面方式启动了spark-shell，就不需要重复启动了）：

```bash
$ cd /usr/local/spark
$ ./bin/spark-shell \
--jars /usr/local/spark/jars/mysql-connector-java-5.1.40/mysql-connector-java-5.1.40-bin.jar \
--driver-class-path  /usr/local/spark/jars/mysql-connector-java-5.1.40/mysql-connector-java-5.1.40-bin.jar
```

上面的命令行中，在一行的末尾加入斜杠\，是为了告诉spark-shell，命令还没有结束。

启动进入spark-shell以后，可以执行以下命令连接数据库，写入数据，程序如下（你可以把下面程序一条条拷贝到spark-shell中执行）：

```scala
import java.util.Properties
import org.apache.spark.sql.types._
import org.apache.spark.sql.Row
 
//下面我们设置两条数据表示两个学生信息
val studentRDD = spark.sparkContext.parallelize(Array("3 Rongcheng M 26","4 Guanhua M 27")).map(_.split(" "))
 
//下面要设置模式信息
val schema = StructType(List(StructField("id", IntegerType, true),StructField("name", StringType, true),StructField("gender", StringType, true),StructField("age", IntegerType, true)))
 
//下面创建Row对象，每个Row对象都是rowRDD中的一行
val rowRDD = studentRDD.map(p => Row(p(0).toInt, p(1).trim, p(2).trim, p(3).toInt))
 
//建立起Row对象和模式之间的对应关系，也就是把数据和模式对应起来
val studentDF = spark.createDataFrame(rowRDD, schema)
 
//下面创建一个prop变量用来保存JDBC连接参数
val prop = new Properties()
prop.put("user", "root") //表示用户名是root
prop.put("password", "hadoop") //表示密码是hadoop
prop.put("driver","com.mysql.jdbc.Driver") //表示驱动程序是com.mysql.jdbc.Driver
 
//下面就可以连接数据库，采用append模式，表示追加记录到数据库spark的student表中
studentDF.write.mode("append").jdbc("jdbc:mysql://localhost:3306/spark", "spark.student", prop)
```

在spark-shell中执行完上述程序后，我们可以看一下效果，看看MySQL数据库中的spark.student表发生了什么变化。请在刚才的另外一个窗口的MySQL命令提示符下面继续输入下面命令：

```mysql
mysql> select * from student;
+------+-----------+--------+------+
| id   | name      | gender | age  |
+------+-----------+--------+------+
|    1 | Xueqian   | F      |   23 |
|    2 | Weiliang  | M      |   24 |
|    3 | Rongcheng | M      |   26 |
|    4 | Guanhua   | M      |   27 |
+------+-----------+--------+------+
4 rows in set (0.00 sec)
```