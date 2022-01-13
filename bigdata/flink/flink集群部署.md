# Flink集群部署

## 一、Standalone模式

搭建Flink集群主要包括以下5个步骤：

- 集群基础配置
- 在集群中安装Java
- 设置SSH无密码登录
- 安装和配置Flink
- 启动和关闭Flink集群

### 1. 集群基础配置

- 所有节点修改主机名

  vim /etc/hostname
  主节点 Master  从节点 Slave1、Slave2

- 所有节点修改 /etc/hosts
  ```bash
  192.168.1.101   Master
  192.168.1.102   Slave1
  192.168.1.103   Slave2
  ```
- 重启系统

### 2. 在集群中安装Java

Flink是运行在JVM上的，因此，需要为集群中的每台机器安装Java环境。对于Flink1.11.2而言，要求使用JDK1.8或者更新的版本。

### 3. 设置SSH无密码登录
忽略文字

### 4. 安装和配置Flink

#### 1）在Master节点上安装Flink

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
export FLNK_HOME=/jrtz/flink
export PATH=$FLINK_HOME/bin:$PATH
```
保存并退出.bashrc文件，然后执行如下命令让配置文件生效：
```bash
source ~/.bashrc
```

#### 2）配置相关文件

- 在Master节点上打开文件 conf/flink-conf.yaml，增加如下两个配置项：

```bash
jobmanager.rpc.address: Master
taskmanager.tmp.dirs: /jrtz/flink/tmp
```

> 注意:  每条配置信息中，冒号后面必须有一个**英文空格**，否则运行时会报错。

- 清空masters文件的原有内容，增加如下一行配置：

```bash
Master:8081
```

- 清空workers文件的原有内容，增加如下3行配置：

```bash
Master
Slave1
Slave2
```

- 把 Master节点的安装文件发送到 Slave节点

```bash
$ cd  /jrtz
$ tar -zcf flink.master.tar.gz ./flink
$ scp  ./flink.master.tar.gz  Slave1:/home/hadoop
$ scp  ./flink.master.tar.gz  Slave2:/home/hadoop
```

- 在Slave1和Slave2节点上分别执行下面同样的操作：

```bash
$ mkdir /jrtz/flink
$ chown -R hadoop:hadoop /jrtz/flink
$ tar -zxf /home/hadoop/flink.master.tar.gz -C /jrtz
```

- 建立 tmp 目录
  在前面配置flink-conf.yaml时，我们设置了临时数据的保存目录“/usr//jrtzk/tmp”。但是，Flink自己不会自动创建这个目录，因此，需要在Master、Slave1和Slave2上分别执行如下命令创建tmp目录并设置权限：

```bash
$ cd /jrtz/flink
$ mkdir tmp && chmod -R 755 ./tmp
```

### 5. 安装和配置Flink

- 在Master节点上执行如下命令启动Flink集群：
  ```bash
  $ cd  /jrtz/flink/
  $ ./bin/start-cluster.sh
  ```

- 启动以后，在Master节点上执行jps命令，可以看到如下信息：
  ```bash
  $ jps
  7265 Jps
  5829 StandaloneSessionClusterEntrypoint
  6153 TaskManagerRunner
  ```

- 在Slave1和Slave2节点上分别执行jps命令，可以看到如下信息：

  ```bash
  $ jps
  4757 TaskManagerRunner
  5639 Jps
  ```

  如果能够看到上述信息，说明集群启动成功。启动成功以后，可以在Master节点上打开浏览器，访问http://master:8081, 就可以通过浏览器查看Flink集群信息。
  Flink 安装包中自带了测试样例，可以在Master、Slave1和Slave2中的任意一个节点上运行WordCount 样例程序来测试 Flink 的运行效果，具体命令如下
  ```bash
  $ cd /jrtz/flink/bin
  $ ./flink run /jrtz/flink/examples/batch/WordCount.jar
  ```
  执行以后，屏幕上就会出现词频统计信息。最后，可以在Master节点上执行如下命令关闭Flink集群：
  ```bash
  $ cd /jrtz/flink && ./bin/stop-cluster.sh
  ```

