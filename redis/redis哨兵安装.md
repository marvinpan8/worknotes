# redis-2.6.17哨兵模式安装

[官网说明文档](https://redis.io/topics/sentinel)

上传redis-2.6.17.tar.gz到指定部署的服务器，解压、编译 

- 如果Linux没有安装GCC，则需要另外安装`yum install gcc`

```bash
tar -zxvf redis-2.6.17.tar.gz
cd redis-2.6.17/src
make
# 若 error: jemalloc/jemalloc.h: No such file or directory解决方法
make MALLOC=libc
make install
# 测试
make test  
# 正常显示
\o/ All tests passed without errors!
# 报错
 You need tcl 8.5 or newer in order to run the Redis test
# 安装tcl
wget http://downloads.sourceforge.net/tcl/tcl8.6.1-src.tar.gz  
tar xzvf tcl8.6.1-src.tar.gz  -C /usr/local/  
cd  /usr/local/tcl8.6.1/unix/  
./configure  
make & make install
# 再次进入redis目录
make test
# 正常显示
\o/ All tests passed without errors!
cp redis-server redis-cli /usr/local/bin/
cd ../
cp redis.conf redis.conf.bak
cp sentinel.conf sentinel.conf.bak
```

如果一台机子上要安装2个redis --- slave则

```bash
mkdir /jrtz/redis/slave1 //创建从服务1目录
mkdir /jrtz/redis/slave2 //创建从服务2目录
cp -r /jrtz/redis/redis-2.6.17 /jrtz/redis/slave1 
cp -r /jrtz/redis/redis-2.6.17 /jrtz/redis/slave2 
```

主节点修改redis.conf

```bash
# 后台运行
# daemonize no
# 修改pid
pidfile /var/run/redis26379.pid
# 修改端口
port 26379
# 绑定固定网卡IP
bind 10.9.1.4
# 添加一个密码
requirepass invest0755
masterauth invest0755
# 修改为你的安装目录，默认当前目录 ./
dir /jrtz/redis/master/

slave-priority 20
-----------------5.0
replica-priority 20
# 2.8版本以上
min-slaves-to-write 1
min-slaves-max-lag 10
----------------------5.0
min-replicas-to-write 1
min-replicas-max-lag 10
# 修改为你的安装目录, 默认stdout，当daemonize=yes将输出/dev/null
# logfile /var/log/redis26379.log
# 关闭rdb快照,通过注释掉所有“save”行来完全禁用保存,删除目录下已经存在的dump.rdb,您可以
#save 900 1
#save 300 10
#save 60 10000
# 关闭AOF，默认
# appendonly no
------------------
client-output-buffer-limit replica 536870912 536870912 0
repl-timeout 1024
#repl-backlog-size 1mb
#repl-backlog-ttl 3600
#延时监控（毫秒）
latency-monitor-threshold 1500
```

从节点增加

```bash
slaveof 10.9.1.4 26379
slave-priority 50
----------------------5.0
replicaof 10.9.1.4 26379 
replica-priority 50
# 修改为你的安装目录，默认当前目录 ./
dir /jrtz/redis/slave1/
```

因为redis采用的是异步复制，在这样的场景下，没有办法避免数据的丢失。然而，你可以通过以下配置来配置redis3和redis1，使得数据不会丢失。

- 2.8版本才有

```
min-slaves-to-write 1
min-slaves-max-lag 10
----------------------
min-replicas-to-write 1
min-replicas-max-lag 10
```

通过上面的配置，当一个redis是master时，如果它不能向至少一个slave写数据(上面的min-slaves-to-write指定了slave的数量)，它将会拒绝接受客户端的写请求。由于复制是异步的，master无法向slave写数据意味着slave要么断开连接了，要么不在指定时间内向master发送同步数据的请求了(上面的min-slaves-max-lag指定了这个时间)。

---

修改sentinel.conf

- 注意：含有mymaster的配置，都必须放置在sentinel monitor mymaster 172.16.48.129 6379 2之后，否则会出现问题
- `sentinel can-failover mymaster yes` v2.6版本才有。指定此哨兵可以为此master开始故障转移。

```bash
#  高版本有bind网卡接口
bind 10.9.1.4
# 高版本有保护模式
protected-mode yes
#这里从服务1的默认端口我们不动 稍后修改从2即可
port 28379
# 高版本有 pidfile
pidfile /var/run/redis-sentinel-28379.pid
#这里配置写上主服务器IP 端口 2个sentinel选举成功后才有效
sentinel monitor mymaster 10.9.1.4 26379 2
# 主节点宕机时间超时时间，标记为S_DOWN，默认30000毫秒,内网设置2秒，需要将配置放在上面一句之下
sentinel down-after-milliseconds mymaster 2000
#主服务器 redis密码
sentinel auth-pass mymaster invest0755
# 设定15秒内master没有活起来，就重新选举主
sentinel failover-timeout mymaster 15000
#在故障转移期间，如果master重新选出来后，其它slave节点能同时并行从新master同步缓存的台数有多少个，显然该值越大，所有slave节点完成同步切换的整体速度越快，但如果此时正好有人在访问这些slave，可能造成读取失败，影响面会更广。最保定的设置为1，只同一时间，只能有一台干这件事，这样其它slave还能继续服务，但是所有slave全部完成缓存更新同步的进程将变慢。。默认1
# sentinel parallel-syncs mymaster 1
```

## 启动redis 和 sentinel 服务

```bash
cd /jrtz/redis/redis-2.6.17 
redis-server redis.conf &
redis-server sentinel.conf --sentinel &
cd ../slave1/ //进入从1服务
redis-server redis26379.conf &
redis-server sentinel.conf --sentinel &
cd ../slave2/ //进入从2服务
redis-server redis26380.conf &
redis-server sentinel.conf --sentinel &

ps -ef |grep redis //查看进程
```

## 开机自启动

### 创建 redis systemd unit 文件

```bash
vim /usr/lib/systemd/system/redismaster.service 
--------------------------------------------------
[Unit]
Description=Redis master Server
After=network.target
Documentation=https://redis.io/documentation

[Service]
ExecStart=/jrtz/redis/master/src/redis-server /jrtz/redis/master/redis.conf
ExecStop=/jrtz/redis/master/src/redis-cli -h 10.9.1.16 -p 26380 -a invest0755 shutdown
Restart=always

[Install]
WantedBy=multi-user.target
```
从节点1 修改2行

```bash
vim /usr/lib/systemd/system/redisslave1.service 
--------------------------------------------------
Description=Redis slave1 Server
....
ExecStart=/jrtz/redis/slave1/src/redis-server /jrtz/redis/slave1/redis.conf
```

从节点2 修改2行

```bash
vim /usr/lib/systemd/system/redisslave2.service 
--------------------------------------------------
Description=Redis slave2 Server
....
ExecStart=/jrtz/redis/slave1/src/redis-server /jrtz/redis/slave2/redis.conf
```



#### 启动

```bash
systemctl daemon-reload && systemctl enable redismaster && systemctl restart redismaster
systemctl daemon-reload && systemctl enable redisslave1 && systemctl restart redisslave1
systemctl daemon-reload && systemctl enable redisslave2 && systemctl restart redisslave2
```

#### 检查

```bash
systemctl status redismaster
systemctl status redisslave1
systemctl status redisslave2
netstat -anp| grep redis
journalctl -u redismaster.service -f -n 500
journalctl -u redisslave1.service -f -n 500
journalctl -u redisslave2.service -f -n 500
```

### 创建 sentinal systemd unit 文件

```bash
vim /usr/lib/systemd/system/redissentinel.service 
--------------------------------------------------
[Unit]
Description=Redis sentinel Server
After=network.target
Documentation=https://redis.io/documentation

[Service]
ExecStart=/jrtz/redis/master/src/redis-server /jrtz/redis/master/sentinel.conf --sentinel
Restart=always

[Install]
WantedBy=multi-user.target
```
从节点1 修改2行

```bash
vim /usr/lib/systemd/system/redissentinel1.service 
--------------------------------------------------
Description=Redis slave1 sentinel Server
....
ExecStart=/jrtz/redis/slave1/src/redis-server /jrtz/redis/slave1/sentinel.conf --sentinel
```

从节点2 修改2行

```bash
vim /usr/lib/systemd/system/redissentinel2.service 
--------------------------------------------------
Description=Redis slave2 sentinel Server
....
ExecStart=/jrtz/redis/slave1/src/redis-server /jrtz/redis/slave1/sentinel.conf --sentinel
```



#### 启动sentinel

```bash
systemctl daemon-reload && systemctl enable redissentinel && systemctl restart redissentinel
systemctl daemon-reload && systemctl enable redissentinel1 && systemctl restart redissentinel1
systemctl daemon-reload && systemctl enable redissentinel2 && systemctl restart redissentinel2
```

#### 检查

```bash
systemctl status redissentinel
netstat -anp| grep redis
---------------------------------------
tcp        0      0 0.0.0.0:28379           0.0.0.0:*               LISTEN      2004/redis-server  
----------------------------------------
journalctl -u redissentinel.service -f -n 500
```

---

## 测试
```bash
redis-cli -h 10.9.1.4 -p 26379 -a invest0755 info replication
redis-cli -h 10.9.1.4 -p 26379 -a invest0755 info Keyspace
redis-cli -h 10.9.1.16 -p 28379 ping
redis-cli -h 10.9.1.16 -p 28379 sentinel ckquorum mymaster
redis-cli -h 10.9.1.16 -p 28379 sentinel get-master-addr-by-name mymaster
redis-cli -h 10.9.1.16 -p 28379 sentinel masters
redis-cli -h 10.9.1.16 -p 28379 sentinel master mymaster
redis-cli -h 10.9.1.16 -p 28379 sentinel slaves mymaster
redis-cli -h 10.9.1.16 -p 28379 sentinel sentinels mymaster
redis-cli -h 10.9.1.4 -p 26379 -a invest0755 DEBUG sleep 30
------------------------------------------------------------
redis-benchmark -h 10.9.1.4 -p 26379 -c 100 -n 100000
redis-benchmark -h 10.9.1.4 -p 26379 -t set,get,LRANGE_100 -n 100000 -q
redis-benchmark -h 10.9.1.4 -p 26379 -n 100000 -q script load "redis.call('set','foo','bar')"
```


## 强制故障转移主节点

对指定<master name>---mymaster 主节点进行强制故障转移（没有和其他Sentinel节点"协商"），当故障转移完成后，其他Sentinel节点按照故障转移的结果更新自身配置，这个命令在Redis Sentinel的日常运维中非常有用。

```bash
redis-cli -h 10.9.1.4 -p 28379 sentinel failover mymaster
```

### 重新启动Master----26379

```bash
systemctl stop redismaster && systemctl stop redissentinel
systemctl start redismaster && systemctl start redissentinel
systemctl restart redismaster && systemctl restart redissentinel
```

###  重新启动Slave1----26379

```bash
systemctl stop redisslave1 && systemctl stop redissentinel1
systemctl start redisslave1 && systemctl start redissentinel1
systemctl restart redisslave1 && systemctl restart redissentinel1
```

### 重新启动Slave2----26380

```bash
systemctl stop redisslave2 && systemctl stop redissentinel2
systemctl start redisslave2 && systemctl start redissentinel2
systemctl restart redisslave2 && systemctl restart redissentinel2
```

## 注意事项

- 当业务场景不需要数据持久化时，关闭所有的持久化方式可以获得最佳的性能以及最大的内存使用量。

- 不要让你的 Redis 所在机器**物理内存**使用超过**实际内存总量**的3/5。

- 根据业务需要选择合适的数据类型，并为不同的应用场景设置相应的紧凑存储参数。

- 按奇数个部署，至少要部署3个，哨兵之间、Redis实例之间物理机独立。

- sentinel monitor master xxx.xxx.xxx.xxx xxxx 1 哨兵的这个配置最好不要配置为1。quorum的值为1意味着只要一个sentinel发现master节点无响应就可以标记为客观下线，从而发起主从切换，quorum最好设置超过sentinel个数的一半向上取整。

- entinel failover-timeout master 900000 //毫秒级； 条件允许的情况下尽可能缩短这个切换间隔吧。

---

### 怎样开启持久化是一个难题。和运维同事探讨了一些方案，这里总结一下供大家参考：
1. **极端情况下可以容忍全量数据丢失**，那么建议master关闭持久化，slave关闭持久化；

2. **极端情况下不能容忍全量数据丢失**，但可以容忍部分数据丢失，如果内存数据集较小且不会增长建议master开启rdb，slave开启rdb；如果数据集很大，或不确定数据集增长趋势，建议master关闭持久化，slave开启rdb；开启rdb需要cpu和磁盘性能保障。如果master关闭持久化，slave开启rdb需要保证slave的rdb不会被master误重启所覆盖，这里提供几种方案：
   - 重启脚本包一层命令先网络请求加载备机备份目录下的rdb文件后再执行start，可以防止误重启，但备机调整部署可能需要调整脚本，主机打开持久化也需要调整脚本。
   - 定时将rdb文件通过网络io传给master节点（文件大比较耗时，文件增长需要考虑定时脚本执行间隔，否则会造成持续的网络io），而且也会有一定数据损失
   - 定时备份Slave的rdb到备份目录，不做任何其他操作，误重启时人工拷贝rdb到master节点（会有一定数据损失）

3. **最大限度需要数据无损**，建议master开启aof，slave开启aof
   - 开启aof需要cpu和磁盘性能保障。开启aof建议fsync同步刷盘使用everysec，自定义脚本在应用空闲时定时做bgrewrite，bgrewrite期间增量数据做缓冲。

目前大部分业务都允许部分数据丢失，为使Redis性能最大化，**关闭了Master持久化，slave开启rdb，为防止误重启对rdb做了5分钟一次备份，保留最近1小时的备份文件，必要时人工copy到master数据目录下恢复数据。**后续硬件性能提升后，看情况再调整持久化机制。

---

#### 故障转移监听做法

这样当发生故障转移时，客户端便可以收到哨兵的通知，从而完成主节点的切换。具体做法是：**利用Redis提供的发布订阅功能，为每一个哨兵节点开启一个单独的线程，订阅哨兵节点的+switch-master频道，当收到消息时，重新初始化连接池。**