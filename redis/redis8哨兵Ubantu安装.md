# redis8哨兵Ubantu安装

- https://download.redis.io/releases/?C=N;O=D

```properties
cd /k0s/redis
tar -zxvf redis-8.6.4.tar.gz

# yum -y install gcc jemalloc
# sudo mv /etc/apt/apt.conf.d/95proxies /etc/apt/apt.conf.d/95proxies.bak
sudo apt update 
sudo apt install -y gcc
sudo apt install -y libjemalloc-dev

cd redis-8.6.4
sudo mkdir -p /ekemp/redis8
sudo chown -R ekemp:ekemp /ekemp
make MALLOC=jemalloc && make install PREFIX=/ekemp/redis8
```

### 配置环境变量

```properties
sudo vim /etc/profile
# 添加一行-------------
export PATH=/ekemp/redis8/bin:$PATH
# 生效执行
source /etc/profile
```

### 修改内核参数

```properties
sudo vim /etc/sysctl.conf
# 修改：
vm.overcommit_memory = 1
net.core.somaxconn = 1024  
#---------
sudo sysctl -p
ulimit -n
ulimit -Sn 131072
sudo vim /etc/security/limits.conf

*       soft    nproc   131072
*       hard    nproc   131072
*       soft    nofile  131072
*       hard    nofile  131072
root    soft    nproc   131072
root    hard    nproc   131072
root    soft    nofile  131072
root    hard    nofile  131072
redis   soft    nofile  131072
redis   hard    nofile  131072
```

## 创建目录
```properties
sudo mkdir /data/redis6380/{conf,data,log} -p
cd /data/redis6380/conf
# sudo vim 6380.conf
```

#### 创建redis用户

```properties
sudo groupadd redis
sudo useradd -g redis redis
sudo chown -R redis:redis /data/redis6380
```

## 配置 Redis 集群

参考官方配置文件 redis.conf、redis-full.conf

#### redis.conf（修改IP）

- sudo vim /data/redis6380/conf/redis.conf

```properties
sudo tee /data/redis6380/conf/redis.conf << EOF
port 6380
bind 10.10.20.201 127.0.0.1
protected-mode no
daemonize no
pidfile "/data/redis6380/data/redis.pid"

dir "/data/redis6380/data"
logfile "/data/redis6380/log/redis.log"
loglevel notice

requirepass zaq1xsw2
masterauth zaq1xsw2

replica-priority 100
min-replicas-to-write 1
min-replicas-max-lag 10

databases 1
maxmemory 6442450944
maxmemory-policy noeviction

appendonly no

slowlog-log-slower-than 10000
slowlog-max-len 128
EOF
```

#### 两个从库配置

基本与主库相同，bindip地址各自对应各自的。需要添加主库同步配置

- sudo vim /data/redis6380/conf/redis.conf

```properties
bind 10.10.20.20X
# 主库为主虚拟机1的地址
replicaof 10.10.20.201 6380
# 优先级90,80
replica-priority 90
```

### 创建系统启动文件

### redis-shutdown

```properties
#!/bin/bash
#
# Wrapper to close properly redis and sentinel
test x"$REDIS_DEBUG" != x && set -x

REDIS_CLI=/ekemp/redis8/bin/redis-cli

# Retrieve service name
SERVICE_NAME="$1"
if [ -z "$SERVICE_NAME" ]; then
   SERVICE_NAME=redis
fi

# Get the proper config file based on service name
CONFIG_FILE="/data/redis6380/conf/$SERVICE_NAME.conf"

# Use awk to retrieve host, port from config file
HOST=`awk '/^[[:blank:]]*bind/ { print $2 }' $CONFIG_FILE | tail -n1`
PORT=`awk '/^[[:blank:]]*port/ { print $2 }' $CONFIG_FILE | tail -n1`
PASS=`awk '/^[[:blank:]]*requirepass/ { print $2 }' $CONFIG_FILE | tail -n1`
SOCK=`awk '/^[[:blank:]]*unixsocket\s/ { print $2 }' $CONFIG_FILE | tail -n1`

# Just in case, use default host, port
HOST=${HOST:-127.0.0.1}
if [ "$SERVICE_NAME" = redis ]; then
    PORT=${PORT:-6379}
else
    PORT=${PORT:-26739}
fi

# Setup additional parameters
# e.g password-protected redis instances
[ -z "$PASS"  ] || ADDITIONAL_PARAMS="-a $PASS"

# shutdown the service properly
if [ -e "$SOCK" ] ; then
	$REDIS_CLI -s $SOCK $ADDITIONAL_PARAMS shutdown
else
	$REDIS_CLI -h $HOST -p $PORT $ADDITIONAL_PARAMS shutdown
fi
```

- chmod +x  redis-shutdown
- cp redis-shutdown  /ekemp/redis8/bin/
- sudo chown -R redis:redis /data/redis6380

#### redis6380.service

```properties
sudo tee /usr/lib/systemd/system/redis6380.service << EOF
[Unit]
Description=Redis Sentinel Database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/ekemp/redis8/bin/redis-server /data/redis6380/conf/redis.conf
ExecStop=/ekemp/redis8/bin/redis-shutdown
Type=simple
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

## 启动

```properties
sudo systemctl daemon-reload 
sudo systemctl enable redis6380 

sudo systemctl start redis6380
sudo systemctl restart redis6380
sudo systemctl stop redis6380
sudo systemctl status redis6380 --full
tail -f -n 500 /data/redis6380/log/redis.log

ps -ef | grep redis
```



---

## 创建哨兵目录

```properties
sudo mkdir /data/redis26380/{data,log} -p
sudo rm -rf /data/redis26380/data/*
sudo rm -rf /data/redis26380/log/*
```


## 配置哨兵（修改IP）

#### sudo vim /data/redis6380/conf/redis-sentinel.conf

```properties
sudo tee /data/redis6380/conf/redis-sentinel.conf << EOF
bind 10.10.20.201
port 26380
protected-mode no
daemonize no
pidfile "/data/redis26380/data/redis-sentinel.pid"
dir "/data/redis26380/data"
logfile "/data/redis26380/log/redis-sentinel.log"
loglevel notice

sentinel monitor ekempmaster 10.10.20.201 6380 2
sentinel auth-pass ekempmaster zaq1xsw2

sentinel down-after-milliseconds ekempmaster 5000
sentinel failover-timeout ekempmaster 15000
sentinel parallel-syncs ekempmaster 2


acllog-max-len 128
sentinel deny-scripts-reconfig yes
SENTINEL resolve-hostnames no
SENTINEL announce-hostnames no
EOF
# 授权
sudo chown -R redis:redis /data/redis6380
```

- **sentinel failover-timeout mymaster 官方默认 3分钟180000**
- **sentinel down-after-milliseconds mymaster 官方默认30秒 30000**

## 启动文件

#### redis-sentinel.service

```properties
sudo tee /usr/lib/systemd/system/redis-sentinel.service << EOF
[Unit]
Description=Redis Sentinel Monitor Database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/ekemp/redis8/bin/redis-sentinel /data/redis6380/conf/redis-sentinel.conf
ExecStop=/ekemp/redis8/bin/redis-shutdown redis-sentinel
Type=simple
User=redis
Group=redis
RuntimeDirectory=redis-sentinel
RuntimeDirectoryMode=0755
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

## 启动

```properties
# 先授权
sudo chown -R redis:redis /data/redis6380
sudo chown -R redis:redis /data/redis26380

sudo systemctl daemon-reload 
sudo systemctl enable redis-sentinel 

sudo systemctl start redis-sentinel
sudo systemctl restart redis-sentinel
sudo systemctl stop redis-sentinel

sudo systemctl status redis-sentinel
tail -f -n 500 /data/redis26380/log/redis-sentinel.log

ps -ef | grep redis
netstat -tunlp|grep 26380
```



---
# 高可用测试

#### 1. 连接redis脚本

```properties
# 主1
redis-cli -h 10.10.20.201 -p 6380 -a zaq1xsw2
# 从2
redis-cli -h 10.10.20.202 -p 6380 -a zaq1xsw2
# 从3
redis-cli -h 10.10.20.203 -p 6380 -a zaq1xsw2
```

#### 2. 同步状态查看

```properties
# 连接完成后输入命令
info replication
# 主库显示如下，即可算完成（包含两个从库ip地址）
# Replication
role:master
connected_slaves:2
slave0:ip=10.10.20.201,port=6380,state=online,offset=188041,lag=1
slave1:ip=10.10.20.202,port=6380,state=online,offset=188041,lag=1

# 从库显示如下，即可算完成
# Replication
role:slave
master_host:10.10.20.201
master_port:6380
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:174548
slave_priority:100
slave_read_only:1
```

#### 3. 主库写入测试同步

```properties
# 主1
set b b
# 从2
keys *
get b
# 从3
keys *
get b
```

#### 4. 从库只读测试

```properties
# 从2
set c c
# result : (error) READONLY You can't write against a read only slave.
# 从3
set c c
# result : (error) READONLY You can't write against a read only slave.
```

#### 5. 成功redis-sentinel日志

```properties
tail -f -n 500 /data/redis26380/log/sentinel.log
```

成功日志，**+slave slave包含两台从库的地址，+sentinel sentinel包含两台哨兵的id**

```properties
 +monitor master mymaster 10.10.20.201 6380 quorum 2
+sdown sentinel 94a67035d30f015eb3fc54c713e9ffca28d2f0cd 10.10.20.202 26380 @ mymaster 10.10.20.201 6380
+sdown sentinel 3225f80ee852dbb7c56c499f34f73917075e8d9f 10.10.20.203 26380 @ mymaster 10.10.20.201 6380
```

#### 6. 成功sentinel的连接状态

```properties
# 主1
$ redis-cli -h 10.10.20.201 -p 26380 INFO Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=10.10.20.201:6380,slaves=2,sentinels=3
# 从2
$ redis-cli -h 10.10.20.202 -p 26380 INFO Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=10.10.20.201:6380,slaves=2,sentinels=3
# 从3
$ redis-cli -h 10.10.20.203 -p 26380 INFO Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=10.10.20.201:6380,slaves=2,sentinels=3
```

#### 7. 高可用测试case

哨兵作为对redis实例的监控，通过选举算法保证哨兵的鲁棒性和高可用，所以哨兵至少要部署3台，符合半数原则，需要5或者，7，超过一半，不包含一半存活的时候，才能够选举出leader，才能进行主从的切换功能。
redis服务，至少需要存活一台，才能保证服务正常运行sentinel 选择新 master 的原则是最近可用 且 数据最新 且 优先级最高 且 活跃最久 ！
**哨兵高可用测试：**分别连接对应的redis服务端，手动停止哨兵，停止主reids服务，看主从是否切换成功。
三哨兵情况：redis实例挂掉两台，剩下一台能够成为主，自动切换

```properties
# 保持三个哨兵进程都存在的情况下
# 1. 三个终端分别连接redis，使用info replication查看当前连接状态：
# 主1
redis-cli -h 10.10.20.201 -p 6380 -a zaq1xsw2 info replication
# 从2
redis-cli -h 10.10.20.202 -p 6380 -a zaq1xsw2 info replication
# 从3
redis-cli -h 10.10.20.203 -p 6380 -a zaq1xsw2 info replication

# 2. 停止当前role:master的对应的redis服务,重新检查状态看是否切换
sudo systemctl stop redis6380
# 虚拟机1的实例转换成为主的redis

# 3. 继续停止剩下两台：role：master的对应的redis服务，重新检查连接状态看是否切换
sudo systemctl stop redis6380
# 虚拟机3的实例转换成为了主的redis
# 切换顺利，实现高可用
```

两哨兵情况：redis实例挂掉两台，剩下一台能够成为主，自动切换

```properties
# 将全部虚拟机的redis + sentinel重新启动
sudo  systemctl start redis6380 && sudo systemctl start redis-sentinel
# 停止虚拟机1的redis-sentinel，重新执行哨兵的案例测试
sudo systemctl stop redis6380 && sudo systemctl stop redis-sentinel
# 分别停止对应master实例的redis，最终剩下一台实例，成为了master，能够自动切换
```

一哨兵情况：redis实例无法主从切换

```properties
# 将全部虚拟机的redis + sentinel重新启动
sudo systemctl start redis6380 && sudo systemctl start redis-sentinel
# 停止虚拟机1和2d的redis-sentinel，重新执行哨兵的案例测试
sudo  systemctl stop redis-sentinel
# 分别停止对应master实例的redis，最终剩下一台实例，无法实现主从切换
```
