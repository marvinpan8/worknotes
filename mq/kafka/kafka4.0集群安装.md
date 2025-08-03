# kafka-4.0集群安装

### jdk21安装

```properties
wget https://download.oracle.com/java/21/latest/jdk-21_linux-x64_bin.rpm
yum install jdk-21_linux-x64_bin.rpm -y
```

vim  /etc/profile

```properties
export JAVA_HOME=/usr/lib/jvm/jdk-21.0.7-oracle-x64
export PATH=$PATH:$JAVA_HOME/bin
```

source /etc/profile

java -version

### 下载kafka安装包

```properties
cd /usr/local
mkdir kafka-cluster
wget https://downloads.apache.org/kafka/4.0.0/kafka_2.13-4.0.0.tgz
tar -zxvf kafka_2.13-4.0.0.tgz
mv kafka_2.13-4.0.0 kafka
rm -f zxvf kafka_2.13-4.0.0.tgz
```

### 修改配置文件

```properties
cd /usr/local/kafka-cluster/kafka/config/
cp server.properties kraft-server.properties
```

### 第一个节点配置

vim kraft-server.properties

```properties
node.id=1

controller.quorum.bootstrap.servers=172.17.0.30:9093,172.17.0.31:9093,172.17.0.32:9093
controller.quorum.voters=1@172.17.0.30:9093,2@172.17.0.31:9093,3@172.17.0.32:9093

listeners=PLAINTEXT://172.17.0.30:9092,CONTROLLER://172.17.0.30:9093
advertised.listeners=PLAINTEXT://172.17.0.30:9092,CONTROLLER://172.17.0.30:9093

log.dirs=/data/kafka/logs

min.insync.replicas=2
num.network.threads=8
num.partitions=3
```

### 第二个节点配置

```properties
node.id=2

controller.quorum.bootstrap.servers=172.17.0.30:9093,172.17.0.31:9093,172.17.0.32:9093
controller.quorum.voters=1@172.17.0.30:9093,2@172.17.0.31:9093,3@172.17.0.32:9093

listeners=PLAINTEXT://172.17.0.31:9092,CONTROLLER://172.17.0.31:9093
advertised.listeners=PLAINTEXT://172.17.0.31:9092,CONTROLLER://172.17.0.31:9093

log.dirs=/data/kafka/logs
```

### 第三个节点配置

```properties
node.id=3

controller.quorum.bootstrap.servers=172.17.0.30:9093,172.17.0.31:9093,172.17.0.32:9093
controller.quorum.voters=1@172.17.0.30:9093,2@172.17.0.31:9093,3@172.17.0.32:9093

listeners=PLAINTEXT://172.17.0.32:9092,CONTROLLER://172.17.0.32:9093
advertised.listeners=PLAINTEXT://172.17.0.32:9092,CONTROLLER://172.17.0.32:9093

log.dirs=/data/kafka/logs
```

### 日志配置

```properties
cd /usr/local/kafka-cluster/kafka
```

在任意节点运行：

```properties
bin/kafka-storage.sh random-uuid
---------------------------------------
rcgozlA-Szyx73QHv6yT4w
```

每台机器执行

```properties
bin/kafka-storage.sh format \
  --cluster-id rcgozlA-Szyx73QHv6yT4w \
  --config config/kraft-server.properties
```

查看

```
ll /data/kafka/logs
cat /data/kafka/logs/meta.properties
```

### 使用JMX监控Kafka

```properties
vim /usr/local/kafka-cluster/kafka/bin/kafka-server-start.sh
在这个字段 export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
下一行加入 export JMX_PORT="9999"
```



### 自启动配置

```properties
cat << EOF | tee /etc/systemd/system/kafka.service
[Unit]
Description=Apache Kafka Server (KRaft mode)
After=network.target

[Service]
Type=simple
Environment="JAVA_HOME=/usr/lib/jvm/jdk-21.0.7-oracle-x64"
ExecStart=/usr/local/kafka-cluster/kafka/bin/kafka-server-start.sh /usr/local/kafka-cluster/kafka/config/kraft-server.properties
ExecStop=/usr/local/kafka-cluster/kafka/bin/kafka-server-stop.sh
Restart=on-failure
User=root
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
EOF
```

### 启动

```bash
systemctl daemon-reload && systemctl enable kafka && systemctl start kafka
systemctl status kafka
journalctl -f -u kafka
```
---

## kafka UI界面

```properties
cat << EOF | tee /etc/systemd/system/KafkaUI.service
[Unit]
Description=KafkaUI Server
After=network.target

[Service]
Type=simple
Environment="JAVA_HOME=/root/jdk1.8.0_351"
ExecStart=/root/jdk1.8.0_351/bin/java -jar /root/KafkaUIByLcc-1.0.jar
ExecStop=/bin/kill -TERM $MAINPID
Restart=on-failure
User=root
LimitNOFILE=100000

[Install]
WantedBy=multi-user.target
EOF
```
### 启动

```properties
systemctl daemon-reload && systemctl start KafkaUI
systemctl status KafkaUI
journalctl -f -u KafkaUI
systemctl stop KafkaUI
```





