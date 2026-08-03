# redis8集群Ubantu安装

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
sudo mkdir /data/redis6379/{conf,data,log} -p
cd /data/redis6379/conf
# sudo vim 6379.conf
```

#### 创建redis用户

```properties
sudo groupadd redis
sudo useradd -g redis redis
sudo chown -R redis:redis /data/redis6379
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
cat << EOF | sudo tee /data/redis6379/conf/6379.conf
port 6379
bind 10.10.20.201 127.0.0.1
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

requirepass "zaq1xsw2"
masterauth "ekempredis"
EOF
```

### 创建系统启动文件

#### redis6379.service

```properties
cat << EOF | sudo tee /usr/lib/systemd/system/redis6379.service
[Unit]
Description=Redis database
After=network.target

[Service]
ExecStart=/ekemp/redis8/bin/redis-server /data/redis6379/conf/6379.conf 
ExecStop=/ekemp/redis8/bin/redis-cli -p 6379 shutdown
Type=simple
LimitNOFILE=65536
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
#### 启动

```properties
sudo systemctl daemon-reload && sudo systemctl enable redis6379 
sudo systemctl start redis6379
sudo systemctl restart redis6379
sudo systemctl stop redis6379
sudo systemctl status redis6379 --full
tail -f -n 500 /data/redis6379/log/redis.log

ps -ef | grep redis
```

### 创建集群

```properties
redis-cli -a zaq1xsw2 --cluster create 10.10.20.201:6379 10.10.20.202:6379 10.10.20.203:6379 
-----------------------------------------
>>> Performing Cluster Check (using node 10.10.20.201:6379)
M: 2603a18ce04e42270cf894da8a758dc0474d39e1 10.10.20.201:6379
   slots:[0-5460] (5461 slots) master
M: fc10687140d0b942ff3529e73c9d92058635253c 10.10.20.202:6379
   slots:[5461-10922] (5462 slots) master
M: 01051ce7740d9e20be8b90c8d7b58cf9b308a22b 10.10.20.203:6379
   slots:[10923-16383] (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

### 验证集群状态

```properties
redis-cli -c -p 6379 -a zaq1xsw2 cluster nodes
redis-cli -c -p 6379 -a zaq1xsw2 cluster info
```



---

# ■■■ 创建备份节点6379

## 创建目录

同上

---
# ■■■ 加入集群

```properties
redis-cli -c -p 6379 -a zaq1xsw2 cluster nodes
# redis-cli -a zaq1xsw2 --cluster help
# 6380
redis-cli -a zaq1xsw2 --cluster add-node 10.10.10.101:6379 10.10.20.201:6379 --cluster-slave --cluster-master-id 2603a18ce04e42270cf894da8a758dc0474d39e1
redis-cli -a zaq1xsw2 --cluster add-node 10.10.10.103:6379 10.10.20.202:6379 --cluster-slave --cluster-master-id fc10687140d0b942ff3529e73c9d92058635253c
redis-cli -a zaq1xsw2 --cluster add-node 10.10.10.105:6379 10.10.20.203:6379 --cluster-slave --cluster-master-id 01051ce7740d9e20be8b90c8d7b58cf9b308a22b
```

### 验证集群状态

```properties
redis-cli -c -p 6379 -a zaq1xsw2 cluster nodes 
redis-cli -c -p 6379 -a zaq1xsw2 cluster info
#----------------------
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:9
cluster_size:3
#----------------------------------------------------------------
redis-cli -c -p 6379 -a zaq1xsw2 --cluster check 10.10.20.201:6379

ll /data/redis6379/conf

# 验证进程
systemctl status redis6379 --full
tail -f -n 500 /data/redis6379/log/redis.log
ps -ef | grep redis
netstat -tunlp | grep redis
```

### 重启命令

```properties
systemctl stop redis6379 && systemctl start redis6379
```



# ■■■集群故障转移

### 手动强制转移
```properties
# 登录从节点端口
redis-cli -c -h 10.10.20.201 -p 6379 -a zaq1xsw2 CLUSTER FAILOVER TAKEOVER
# 自动修复Slots分配不一致
redis-cli -a zaq1xsw2 --cluster fix 10.10.20.201:6379

# 删除节点,需节点ID
redis-cli -a zaq1xsw2 --cluster del-node 10.10.10.101:6379 54c77b8fabaae76b57c2cec835d7b06ca9a585d8
# 重新添加节点
redis-cli -a zaq1xsw2 --cluster add-node 10.10.10.101:6379 10.10.20.201:6379 --cluster-slave --cluster-master-id ac013ec387ca77d45d586db55fa211e60a7a53e6
```

