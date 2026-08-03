# redis8集群Centos安装

```properties
cd /k0s/redis
tar -zxvf redis-8.0.3.tar.gz

yum -y install gcc jemalloc

cd redis-8.0.3
make MALLOC=jemalloc && make install PREFIX=/k0s/redis/redis8
```

### 配置环境变量

```properties
vim /etc/profile
# 添加一行-------------
export PATH=/k0s/redis/redis8/bin:$PATH
# 生效执行
source /etc/profile
```

### 修改内核参数

```properties
$ vim /etc/sysctl.conf
# 修改：
vm.overcommit_memory = 1
net.core.somaxconn = 1024  
#---------
$ sysctl -p
$ ulimit -n
$ ulimit -Sn 131072
$ vim /etc/security/limits.conf

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
mkdir /data/redis6379/{conf,data,log} -p
cd /data/redis6379/conf
vim 6379.conf
```

#### 创建redis用户

```properties
groupadd redis
useradd -g redis redis
chown -R redis:redis /data/redis6379
```

## 配置 Redis 集群

参考官方配置文件 redis.conf、redis-full.conf

#### 6379.conf（修改IP）

- **判断节点超时时间默认15秒：**cluster-node-timeout 15000
- **允许在部分 Slot 不可用的情况下集群可用**：cluster-require-full-coverage no
- **可用性优先，强制转移设置为0**：cluster-replica-validity-factor 0
- **设置主节点密码：**masterauth "admin"
- **数据库只能1个：**databases 1
- **每个主节点最少从节点数量：**cluster-migration-barrier 1（默认）
- **允许副本自动迁移给其他主节点:**  cluster-allow-replica-migration yes默认）

```properties
cat << EOF | tee /data/redis6379/conf/6379.conf
port 6379
bind 172.17.0.40 127.0.0.1
pidfile "/data/redis6379/data/redis.pid"
daemonize no
cluster-enabled yes
cluster-config-file /data/redis6379/conf/nodes-6379.conf
cluster-node-timeout 15000
cluster-require-full-coverage no
cluster-replica-validity-factor 0

dir "/data/redis6379/data"
loglevel notice
logfile "/data/redis6379/log/redis.log"

databases 1
maxmemory 3gb
maxmemory-policy noeviction

appendonly no
appendfilename "appendonly.aof"
slowlog-log-slower-than 10000
slowlog-max-len 128

requirepass "admin"
masterauth "admin"
EOF
```

### 创建系统启动文件

#### redis6379.service

```properties
cat << EOF | tee /usr/lib/systemd/system/redis6379.service
[Unit]
Description=Redis database
After=network.target

[Service]
ExecStart=/k0s/redis/redis8/bin/redis-server /data/redis6379/conf/6379.conf 
ExecStop=/k0s/redis/redis8/bin/redis-cli -p 6379 shutdown
Type=simple
LimitNOFILE=65536
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
#### 启动

```properties
systemctl daemon-reload  && systemctl enable redis6379 && systemctl restart redis6379
systemctl stop redis6379
systemctl status redis6379 --full
tail -f -n 500 /data/redis6379/log/redis.log

ps -ef | grep redis
```

### 创建集群

```properties
redis-cli -a admin --cluster create 172.17.0.40:6379 172.17.0.41:6379 172.17.0.42:6379 
```

### 验证集群状态

```properties
redis-cli -c -p 6379 -a admin cluster nodes
redis-cli -c -p 6379 -a admin cluster info
```



---

# ■■■ 创建备份节点6380

## 创建目录
```properties
mkdir /data/redis6380/{conf,data,log} -p
```

#### 6380.conf（修改IP）

- **判断节点超时时间默认15秒：**cluster-node-timeout 15000
- **允许在部分 Slot 不可用的情况下集群可用**：cluster-require-full-coverage no
- **可用性优先，强制转移设置为0**：cluster-replica-validity-factor 0
- **设置主节点密码：**masterauth "admin"
- **数据库只能1个：**databases 1
- **每个主节点最少从节点数量：**cluster-migration-barrier 1（默认）
- **允许副本自动迁移给其他主节点:**  cluster-allow-replica-migration yes默认）

```properties
cat << EOF | tee /data/redis6380/conf/6380.conf
port 6380
bind 172.17.0.42 127.0.0.1
pidfile "/data/redis6380/data/redis.pid"
daemonize no
cluster-enabled yes
cluster-config-file /data/redis6380/conf/nodes-6380.conf
cluster-node-timeout 15000
cluster-require-full-coverage no
cluster-replica-validity-factor 0

dir "/data/redis6380/data"
loglevel notice
logfile "/data/redis6380/log/redis.log"

databases 1
maxmemory 3gb
maxmemory-policy noeviction

appendonly no
appendfilename "appendonly.aof"
slowlog-log-slower-than 10000
slowlog-max-len 128

requirepass "admin"
masterauth "admin"
EOF
```

#### redis6380.service

```properties
cat << EOF | tee /usr/lib/systemd/system/redis6380.service
[Unit]
Description=Redis database
After=network.target

[Service]
ExecStart=/k0s/redis/redis8/bin/redis-server /data/redis6380/conf/6380.conf 
ExecStop=/k0s/redis/redis8/bin/redis-cli -p 6380 shutdown
Type=simple
LimitNOFILE=65536
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
#### 启动

```properties
systemctl daemon-reload  && systemctl enable redis6380 && systemctl restart redis6380

tail -f -n 500 /data/redis6380/log/redis.log
```

---

# ■■■ 创建备份节点6381

## 创建目录
```properties
mkdir /data/redis6381/{conf,data,log} -p
```

#### 6381.conf（修改IP）

- **判断节点超时时间默认15秒：**cluster-node-timeout 15000
- **允许在部分 Slot 不可用的情况下集群可用**：cluster-require-full-coverage no
- **可用性优先，强制转移设置为0**：cluster-replica-validity-factor 0
- **设置主节点密码：**masterauth "admin"
- **数据库只能1个：**databases 1
- **每个主节点最少从节点数量：**cluster-migration-barrier 1（默认）
- **允许副本自动迁移给其他主节点:**  cluster-allow-replica-migration yes默认）

```properties
cat << EOF | tee /data/redis6381/conf/6381.conf
port 6381
bind 172.17.0.40 127.0.0.1
pidfile "/data/redis6381/data/redis.pid"
daemonize no
cluster-enabled yes
cluster-config-file /data/redis6381/conf/nodes-6381.conf
cluster-node-timeout 15000
cluster-require-full-coverage no
cluster-replica-validity-factor 0

dir "/data/redis6381/data"
loglevel notice
logfile "/data/redis6381/log/redis.log"

databases 1
maxmemory 3gb
maxmemory-policy noeviction

appendonly no
appendfilename "appendonly.aof"
slowlog-log-slower-than 10000
slowlog-max-len 128

requirepass "admin"
masterauth "admin"
EOF
```

#### redis6381.service

```properties
cat << EOF | tee /usr/lib/systemd/system/redis6381.service
[Unit]
Description=Redis database
After=network.target

[Service]
ExecStart=/k0s/redis/redis8/bin/redis-server /data/redis6381/conf/6381.conf 
ExecStop=/k0s/redis/redis8/bin/redis-cli -p 6381 shutdown
Type=simple
LimitNOFILE=65536
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
#### 启动

```properties
systemctl daemon-reload  && systemctl enable redis6381 && systemctl start redis6381
```



---
# ■■■ 加入集群

```properties
redis-cli -c -p 6379 -a admin cluster nodes
# redis-cli -a admin --cluster help
# 6380
redis-cli -a admin --cluster add-node 172.17.0.40:6380 172.17.0.41:6379 --cluster-slave --cluster-master-id a6044d9c8448f8302d67cd9898f8074a3c84ba3d
redis-cli -a admin --cluster add-node 172.17.0.41:6380 172.17.0.42:6379 --cluster-slave --cluster-master-id 4c9dae7f4819ae18b95e2ec64eee178fc907a46c
redis-cli -a admin --cluster add-node 172.17.0.42:6380 172.17.0.40:6379 --cluster-slave --cluster-master-id ac013ec387ca77d45d586db55fa211e60a7a53e6
# 6381
redis-cli -a admin --cluster add-node 172.17.0.42:6381 172.17.0.41:6379 --cluster-slave --cluster-master-id a6044d9c8448f8302d67cd9898f8074a3c84ba3d
redis-cli -a admin --cluster add-node 172.17.0.40:6381 172.17.0.42:6379 --cluster-slave --cluster-master-id 4c9dae7f4819ae18b95e2ec64eee178fc907a46c
redis-cli -a admin --cluster add-node 172.17.0.41:6381 172.17.0.40:6379 --cluster-slave --cluster-master-id ac013ec387ca77d45d586db55fa211e60a7a53e6
```

### 验证集群状态

```properties
redis-cli -c -p 6379 -a admin cluster nodes 
redis-cli -c -p 6379 -a admin cluster info
#----------------------
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:9
cluster_size:3
#----------------------------------------------------------------
redis-cli -c -p 6379 -a admin --cluster check 172.17.0.41:6379

ll /data/redis6379/conf
ll /data/redis6380/conf

# 验证进程
systemctl status redis6379 --full
systemctl status redis6380 --full
systemctl status redis6381 --full
tail -f -n 500 /data/redis6379/log/redis.log
tail -f -n 500 /data/redis6380/log/redis.log
tail -f -n 500 /data/redis6381/log/redis.log
ps -ef | grep redis
netstat -tunlp | grep redis
```

### 重启命令

```properties
systemctl stop redis6379 && systemctl start redis6379
systemctl stop redis6380 && systemctl start redis6380
systemctl stop redis6381 && systemctl start redis6381
```



# ■■■集群故障转移

### 手动强制转移
```properties
# 登录从节点端口
redis-cli -c -h 172.17.0.42 -p 6379 -a admin CLUSTER FAILOVER TAKEOVER
# 自动修复Slots分配不一致
redis-cli -a admin --cluster fix 172.17.0.41:6381
# 删除节点,需节点ID
redis-cli -a admin --cluster del-node 172.17.0.41:6381 54c77b8fabaae76b57c2cec835d7b06ca9a585d8
# 重新添加节点
redis-cli -a admin --cluster add-node 172.17.0.41:6381 172.17.0.40:6379 --cluster-slave --cluster-master-id ac013ec387ca77d45d586db55fa211e60a7a53e6
```

