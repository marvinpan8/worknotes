# Linux部署

## 硬件选择

- Linux Kernel 建议3.10以上的内核
- BE推荐16核64GB以上，FE推荐8核16GB以上。
- 磁盘可以使用HDD或者SSD。
- CPU必须支持AVX2指令集，`cat /proc/cpuinfo |grep avx2` 确认有输出即可，如果没有支持，建议更换机器，StarRocks的向量化技术需要CPU指令集支持才能发挥更好的效果。
- 网络需要万兆网卡和万兆交换机。

## 环境准备

- CPU需要支持AVX2指令集， 检查

```bash
$ cat /proc/cpuinfo |grep avx2
```

- 安装 jdk1.8+

```bash
tar -zxvf jdk-8u191-linux-x64.tar.gz
rm -f jdk-8u191-linux-x64.tar.gz
mv jdk1.8.0_191 /jrtz/jdk
```

- 配置环境变量，在末尾添加：

vim /etc/profile

```bash
#jdk config
export JAVA_HOME=/jrtz/jdk
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JRE_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
```

让配置立即生效：

```bash
source /etc/profile
# 检查是否配置成功。
java -version
```

## StarRocks 安装

- 官网下载安装包 https://www.starrocks.com/zh-CN/download/community
- 分发包所有机器，所有机器进行安装

- 创建用户，解压

```bash
$ useradd StarRocks
$ mv StarRocks-1.19.1.tar.gz /home/StarRocks/
$ chown StarRocks:StarRocks /home/StarRocks/StarRocks-1.19.1.tar.gz
$ su - StarRocks
$ tar -zxvf StarRocks-1.19.1.tar.gz
$ rm -f StarRocks-1.19.1.tar.gz
$ mv StarRocks-1.19.1 StarRocks
```

## 部署FE

### FE的基本配置

- FE的配置文件为StarRocks-XX-1.0.0/fe/conf/fe.conf, 默认配置已经足以启动集群

- 有经验的用户可以查看手册的系统配置章节, 为生产环境定制配置

- 为了让用户更好的理解集群的工作原理, 此处只列出基础配置

### FE单实例部署

```bash
cd StarRocks/fe
```

#### 第一步: 定制配置文件conf/fe.conf：

- STARROCKS_HOME 变量已经在  bin/start_fe.sh 声明

- 指定主机网卡 priority_networks  = 192.168.10.212/24
- 可以根据FE内存大小调整 -Xmx4096m，为了避免GC建议16G以上，StarRocks的元数据都在内存中保存。
- 可以在 http://fe_leader_ip:8030/variable 中查看所有的有效配置

```bash
JAVA_OPTS = "-Xmx4096m -XX:+UseMembar -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=7 -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -Xloggc:$STARROCKS_HOME/log/fe.gc.log.$DATE"

priority_networks = 192.168.10.212/24
```
#### 第二步: 创建元数据目录

1.19.x及以前的版本需要使用

```bash
mkdir -p doris-meta
```

新版
```bash
mkdir -p meta
```

#### 第三步: 启动FE进程

```bash
bin/start_fe.sh --daemon
```

#### 第四步: 确认启动FE是否成功

- 查看日志log/fe.log确认

  tail -f -n 500 log/fe.log

```plaintext
2020-03-16 20:32:14,686 INFO 1 [FeServer.start():46] thrift server started.
2020-03-16 20:32:14,696 INFO 1 [NMysqlServer.start():71] Open mysql server success on 9030
2020-03-16 20:32:14,696 INFO 1 [QeService.start():60] QE service start.
2020-03-16 20:32:14,825 INFO 76 [HttpServer$HttpServerThread.run():210] HttpServer started with port 8030
...
```

- 如果FE启动失败，可能是由于端口号被占用，修改配置文件conf/fe.conf中的端口号http_port。
- 使用 jps 命令查看java进程确认"**StarRocksFe**"存在.
- 使用浏览器访问8030端口, 打开StarRocks的WebUI, 用户名为root, 密码为空.

### 使用MySQL客户端访问FE

#### 第一步: 安装mysql客户端

(如果已经安装，可忽略此步)：

下载mysql-client 相关rpm包

```bash
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-client-5.7.36-1.el7.x86_64.rpm
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-libs-5.7.36-1.el7.x86_64.rpm
wget https://repo.mysql.com/yum/mysql-5.7-community/el/7/x86_64/mysql-community-common-5.7.36-1.el7.x86_64.rpm
```
检查是否已经安装 mariadb 包，强制卸载它。否则会冲突
```bash
$ rpm -qa | grep mariadb
mariadb-libs-5.5.60-1.el7_5.x86_64
# 强制删除 --nodeps
$ rpm -e --nodeps mariadb-libs-5.5.60-1.el7_5.x86_64
```

安装
```bash
rpm -Uvh mysql-community-common-5.7.36-1.el7.x86_64.rpm
rpm -Uvh mysql-community-libs-5.7.36-1.el7.x86_64.rpm
rpm -Uvh mysql-community-client-5.7.36-1.el7.x86_64.rpm
```

#### 第二步: 使用mysql客户端连接

```bash
mysql -h 127.0.0.1 -P9030 -uroot
```

注意：这里默认root用户密码为空，端口为fe/conf/fe.conf中的query_port配置项，默认为9030

#### 第三步: 查看FE状态

```mysql
mysql> SHOW PROC '/frontends'\G
*************************** 1. row ***************************
             Name: 192.168.10.212_9010_1636440107329
               IP: 192.168.10.212
         HostName: 192.168.10.212
      EditLogPort: 9010
         HttpPort: 8030
        QueryPort: 9030
          RpcPort: 9020
             Role: FOLLOWER
         IsMaster: true
        ClusterId: 937501450
             Join: true
            Alive: true
ReplayedJournalId: 98
    LastHeartbeat: 2021-11-09 14:47:13
         IsHelper: true
           ErrMsg: 
1 row in set (0.04 sec)
```

- Role为 **FOLLOWER** 说明这是一个能参与选主的 FE；

- **IsMaster** 为 true，说明该FE当前为主节点。
- 如果MySQL客户端连接不成功，请查看log/fe.warn.log日志文件，确认问题。由于是初次启动，如果在操作过程中遇到任何意外问题，都可以删除并重新创建FE的元数据目录，再从头开始操作。

### FE的高可用集群部署

- FE节点之间的时钟相差不能超过5s, 使用NTP协议校准时间
- 一台机器上只可以部署单个FE节点。所有FE节点的http_port需要相同
- Follower FE(包括Master)的数量必须为奇数，建议部署3个，组成高可用(HA)模式即可。
- 当 FE 处于高可用部署时（1个Master，2个Follower），建议通过增加 Observer FE 来扩展 FE 的读服务能力。当然也可以继续增加 Follower FE，但几乎是不必要的。
- 通常一个 FE 节点可以应对 10-20 台 BE 节点。建议总的 FE 节点数量在 10 个以下。而3个即可满足绝大部分需求。

#### 第一步: 分发二进制和配置文件

配置文件和单实例情形相同.

#### 第二步: FE扩容缩容

FE扩容

部署好FE节点，启动完成服务后，通过命令扩容FE节点。

```sql
alter system add follower "fe_host:edit_log_port";
alter system add observer "fe_host:edit_log_port";
```

缩容和扩容命令类似

```sql
alter system drop follower "fe_host:edit_log_port";
alter system drop observer "fe_host:edit_log_port";
```

- host为机器的IP

- port为edit_log_port，默认为9010。

#### 第三步: 启动从节点

其他从节点执行

```bash
./bin/start_fe.sh --helper 192.168.10.222:9010 --daemon
```

当FE再次启动时，无须指定--helper参数，因为FE已经将其他FE的配置信息存储于本地目录, 因此可直接启动：

```shell
./bin/start_fe.sh --daemon
```

#### 第四步: 查看集群状态

```plaintext
$ mysql -h 127.0.0.1 -P9030 -uroot
---------------------------------
mysql> SHOW PROC '/frontends'\G

********************* 1. row **********************
    Name: 172.26.108.172_9010_1584965098874
      IP: 172.26.108.172
HostName: starrocks-sandbox01
......
    Role: FOLLOWER
IsMaster: true
......
   Alive: true
......
********************* 2. row **********************
    Name: 172.26.108.174_9010_1584965098874
      IP: 172.26.108.174
HostName: starrocks-sandbox02
......
    Role: FOLLOWER
IsMaster: false
......
   Alive: true
......
********************* 3. row **********************
    Name: 172.26.108.175_9010_1584965098874
      IP: 172.26.108.175
HostName: starrocks-sandbox03
......
    Role: FOLLOWER
IsMaster: false
......
   Alive: true
......
3 rows in set (0.05 sec)
```

节点的Alive显示为true则说明添加节点成功。以上例子中，

172.26.108.172_9010_1584965098874 为主FE节点。

### 删除重来

```bash
rm -rf plugins temp_dir log doris-meta && mkdir doris-meta
```

## 部署BE

### BE的基本配置

- BE的配置文件为StarRocks-XX-1.0.0/be/conf/be.conf, 默认配置已经足以启动集群, 不建议初尝用户修改配置, 
- 有经验的用户可以查看手册的系统配置章节, 为生产环境定制配置. 
- 为了让用户更好的理解集群的工作原理, 此处只列出基础配置.

### BE部署

用户可使用下面命令添加BE到StarRocks集群, 一般至少部署3个BE实例, 每个实例的添加步骤相同.

```bash
cd StarRocks-XX-1.0.0/be
```

#### 定制配置文件conf/be.conf

```bash
priority_networks = 192.168.10.212/24
```

#### 第一步: 创建数据目录

```bash
mkdir -p storage
```

#### 第二步: 添加BE节点

 通过mysql客户端添加BE节点：

```bash
$ mysql -h 127.0.0.1 -P9030 -uroot
# mysql> ALTER SYSTEM ADD BACKEND "host:port";
mysql> ALTER SYSTEM ADD BACKEND "192.168.10.222:9050";
```

这里IP地址为和priority_networks设置匹配的IP，port:  heartbeat_service_port，默认为9050

如出现错误，需要删除BE节点，应用下列命令：
```bash
alter system decommission backend "be_host:be_heartbeat_service_port";
alter system dropp backend "be_host:be_heartbeat_service_port";
```

具体参考[扩容缩容](https://docs.starrocks.com/zh-cn/1.19/administration/Scale_up_down)。

#### 第三步: 启动BE

```shell
bin/start_be.sh --daemon
```

#### 第四步: 查看BE状态

确认BE就绪:

```sql
mysql> SHOW PROC '/backends'\G

********************* 1. row **********************
            BackendId: 10002
              Cluster: default_cluster
                   IP: 172.16.139.24
             HostName: starrocks-sandbox01
        HeartbeatPort: 9050
               BePort: 9060
             HttpPort: 8040
             BrpcPort: 8060
        LastStartTime: 2020-03-23 20:19:07
        LastHeartbeat: 2020-03-23 20:34:49
                Alive: true
 SystemDecommissioned: false
ClusterDecommissioned: false
            TabletNum: 0
     DataUsedCapacity: .000
        AvailCapacity: 327.292 GB
        TotalCapacity: 450.905 GB
              UsedPct: 27.41 %
       MaxDiskUsedPct: 27.41 %
               ErrMsg:
              Version:
1 row in set (0.01 sec)
```

如果isAlive为true，则说明BE正常接入集群。如果BE没有正常接入集群，请查看log目录下的be.WARNING日志文件确定原因。

如果日志中出现类似以下的信息，说明priority_networks的配置存在问题。

```plaintext
W0708 17:16:27.308156 11473 heartbeat\_server.cpp:82\] backend ip saved in master does not equal to backend local ip127.0.0.1 vs. 172.16.179.26
```

此时需要，先用以下命令drop掉原来加进去的be，然后重新以正确的IP添加BE。

```sql
mysql> ALTER SYSTEM DROPP BACKEND "192.168.10.212:9050";
```

由于是初次启动，如果在操作过程中遇到任何意外问题，都可以删除并重新创建storage目录，再从头开始操作。



## 部署Broker

- broker是额外的组件，如果不需要和hdfs交互，不部署也可以
- 若需要与hdfs交互，建议和BE数保持一致

配置文件为apache_hdfs_broker/conf/apache_hdfs_broker.conf

> 注意：Broker没有也不需要priority_networks参数，Broker的服务默认绑定在0.0.0.0上，只需要在ADD BROKER时，填写正确可访问的Broker IP即可。

如果有特殊的hdfs配置，复制线上的hdfs-site.xml到conf目录下

启动：

```shell
./apache_hdfs_broker/bin/start_broker.sh --daemon
```

添加broker节点到集群中：

```bash
$ mysql -h 127.0.0.1 -P9030 -uroot
MySQL> ALTER SYSTEM ADD BROKER broker1 "192.168.10.222:8000";
```

查看broker状态：

```plaintext
MySQL> SHOW PROC "/brokers"\G
*************************** 1. row ***************************
          Name: broker1
            IP: 172.16.139.24
          Port: 8000
         Alive: true
 LastStartTime: 2020-04-01 19:08:35
LastUpdateTime: 2020-04-01 19:08:45
        ErrMsg: 
1 row in set (0.00 sec)
```

Alive为true代表状态正常。

---

## 使用MySQL客户端访问StarRocks

### Root用户登录

使用MySQL客户端连接某一个FE实例的query_port(9030), StarRocks内置root用户，密码默认为空：

```shell
mysql -h 192.168.10.212 -P9030 -u root
```