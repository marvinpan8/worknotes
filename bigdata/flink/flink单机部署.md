# flink 单机部署

## 单机部署
- [Flink官网](https://flink.apache.org/zh/downloads.html)下载Flink安装文件 flink-1.11.4-bin-scala_2.11.tgz，scala版本 2.11

```bash
$ cd /jrtz/flink
$ tar -zxvf flink-1.11.4-bin-scala_2.11.tgz
$ mv ./flink-1.11.4 ./flink
$ chown -R hadoop:hadoop ./flink  # hadoop是当前登录Linux系统的用户名
```
- 添加环境变量

vim ~/.bashrc
```bash
export FLNK_HOME=/usr/local/flink
export PATH=$FLINK_HOME/bin:$PATH
```
保存并退出.bashrc文件，然后执行如下命令让配置文件生效：
```bash
source ~/.bashrc
```
- 启动Flink：

```bash
$ bin/start-cluster.sh
```

- 使用jps命令查看进程：

  ```bash
  [hadoop@vmsrv-010-225 flink]$ jps
  21984 Jps
  21543 StandaloneSessionClusterEntrypoint
  21869 TaskManagerRunner
  ```
- 页面访问 
Flink的JobManager同时会在8081端口上启动一个Web前端，可以在浏览器中输入“http://localhost:8081”来访问

- 测试

  Flink安装包中自带了测试样例，这里可以运行WordCount样例程序来测试Flink的运行效果，具体命令如下：

  ```bash
  $ cd /jrtz/flink/bin
  $ ./flink run /jrtz/flink/examples/batch/WordCount.jar
  Executing WordCount example with default input data set.
  Use --input to specify file input.
  Printing result to stdout. Use --output to specify output path.
  Job has been submitted with JobID c8181ecfe182d01ff9a041ccf3ae7fa1
  Program execution finished
  Job with JobID c8181ecfe182d01ff9a041ccf3ae7fa1 has finished.
  Job Runtime: 574 ms
  Accumulator Results: 
  - f147ed1969ec11f77176df0876037b62 (java.util.ArrayList) [170 elements]
  
  
  (a,5)
  (action,1)
  (after,1)
  (against,1)
  (all,2)
  (and,12)
  (arms,1)
  (arrows,1)
  (awry,1)
  ……
  ```
## Flink Scala Shell

- 启动Scala Shell环境：

  注意先关闭Flink:  stop-cluster.sh
```bash
$ cd  /jrtz/flink
$ ./bin/start-scala-shell.sh local
```

- 正常进入“scala>”命令提示符状态

  ```bash
  $ ./bin/start-scala-shell.sh local
  Starting Flink Shell:
  Starting local Flink cluster (host: localhost, port: 8081).
  Connecting to Flink cluster (host: localhost, port: 8081).
  
  NOTE: Use the prebound Execution Environments and Table Environment to implement batch or streaming programs.
  
    Batch - Use the 'benv' and 'btenv' variable
  
      * val dataSet = benv.readTextFile("/path/to/data")
      * dataSet.writeAsText("/path/to/output")
      * benv.execute("My batch program")
      *
      * val batchTable = btenv.fromDataSet(dataSet)
      * btenv.registerTable("tableName", batchTable)
      * val result = btenv.sqlQuery("SELECT * FROM tableName").collect
      HINT: You can use print() on a DataSet to print the contents or collect()
      a sql query result back to the shell.
  
    Streaming - Use the 'senv' and 'stenv' variable
  
      * val dataStream = senv.fromElements(1, 2, 3, 4)
      * dataStream.countWindowAll(2).sum(0).print()
      *
      * val streamTable = stenv.fromDataStream(dataStream, 'num)
      * val resultTable = streamTable.select('num).where('num % 2 === 1 )
      * resultTable.toAppendStream[Row].print()
      * senv.execute("My streaming program")
      HINT: You can only print a DataStream to the shell in local mode.
        
  
  scala>
  ```

  