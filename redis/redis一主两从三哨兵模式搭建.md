# redis（一主两从三哨兵模式搭建）记录
https://www.cnblogs.com/fly-piglet/p/9836314.html
### 目的:

让看看这篇文章的的人能够知道：软件架构、软件的安装、配置、基本运维的操作、高可用测试、也包含我自己，能够节省对应的时间。

### 软件架构：

生产环境使用三台服务器搭建redis哨兵集群，3个redis实例（1主2从）+ 3个哨兵实例。生产环境能够保证在哨兵存活两台的情况下，只有一台redis能够继续提供服务（一主两从三哨兵）

| 主虚拟机1     | 从虚拟机2     | 从虚拟机3     |
| ------------- | ------------- | ------------- |
| 172.16.48.129 | 172.16.48.130 | 172.16.48.131 |

#### 软件安装：分别在三台机器上通过yum进行redis的下载和安装以及开机启动

```
# 添加软件安装源
yum install epel-release
# 安装redis
yum install redis -y
# 启动redis、启动redis哨兵
systemctl start redis
systemctl start redis-sentinel
# 允许开机启动
systemctl enable redis
systemctl enable redis-sentinel
# 之后进行配置修改：为哨兵集群，重启启动服务
```

#### /etc/redis.conf（主库配置）

```
# 修改redis配置文件：/etc/redis.conf
# 1. 修改绑定ip为服务器内网ip地址，做绑定，三台各自填写各自的ip地址
bind 172.16.48.129
# 2. 保护模式修改为否，允许远程连接
protected-mode no
# 4. 设定密码
requirepass "123456789"
# 5. 设定主库密码与当前库密码同步，保证从库能够提升为主库
masterauth "123456789"
# 6. 打开AOF持久化支持
appendonly yes
```

#### /etc/redis.conf（两个从库配置）

基本配置和主库相同，bindip地址各自对应各自的。
需要添加主库同步配置

```
# 主库为主虚拟机1的地址
slaveof 172.16.48.129 6379
```

#### /etc/redis-sentinel.conf（哨兵配置）

```
# 修改redis-sentinel配置文件：/etc/redis-sentinel.conf
# 1. 绑定的地址
bind 172.19.131.247
# 2. 保护模式修改为否，允许远程连接
protected-mode no
# 3. 设定sentinel myid 每个都不一样,使用yum安装的时候，直接就生成了
sentinel myid 04d9d3fef5508f60498ac014388571e719188527
# 4. 设定监控地址，为对应的主redis库的内网地址
sentinel monitor mymaster 172.16.48.129 6379 2
# 5. 设定5秒内没有响应，说明服务器挂了,需要将配置放在sentinel monitor master 127.0.0.1 6379 1下面
sentinel down-after-milliseconds mymaster 5000
# 6. 设定15秒内master没有活起来，就重新选举主
sentinel failover-timeout mymaster 15000
# 7. 表示如果master重新选出来后，其它slave节点能同时并行从新master同步缓存的台数有多少个，显然该值越大，所有slave节点完成同步切换的整体速度越快，但如果此时正好有人在访问这些slave，可能造成读取失败，影响面会更广。最保定的设置为1，只同一时间，只能有一台干这件事，这样其它slave还能继续服务，但是所有slave全部完成缓存更新同步的进程将变慢。
sentinel parallel-syncs mymaster 2
# 8. 主数据库密码,需要将配置放在sentinel monitor master 127.0.0.1 6379 1下面
sentinel auth-pass mymaster 123456789
```

> 注意：含有mymaster的配置，都必须放置在sentinel monitor mymaster 172.16.48.129 6379 2之后，否则会出现问题

#### 3. 重新启动

```
# 启动需要按照Master->Slave->Sentinel的顺序进行启动
# 启动redis
systemctl restart redis
# 启动redis哨兵
systemctl restart redis-sentinel
```

### 高可用测试：

#### 1. 连接redis脚本

```
# 主虚拟机1
redis-cli -h 172.16.48.129 -p 6379 -a 123456789
# 从虚拟机2
redis-cli -h 172.16.48.130 -p 6379 -a 123456789
# 从虚拟机3
redis-cli -h 172.16.48.131 -p 6379 -a 123456789
```

#### 2. 同步状态查看

```
# 连接完成后输入命令
info replication
# 主库显示如下，即可算完成（包含两个从库ip地址）
# Replication
role:master
connected_slaves:2
slave0:ip=172.16.48.131,port=6379,state=online,offset=188041,lag=1
slave1:ip=172.16.48.130,port=6379,state=online,offset=188041,lag=1
master_repl_offset:188041
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:2
repl_backlog_histlen:188040
# 从库显示如下，即可算完成
# Replication
role:slave
master_host:172.16.48.129
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:174548
slave_priority:100
slave_read_only:1
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
```

#### 3. 主库写入测试同步

```
# 主虚拟机1
set b b
# 从虚拟机2
keys *
get b
# 从虚拟机3
keys *
get b
```

#### 4. 从库只读测试

```
# 从虚拟机2
set c c
# result : (error) READONLY You can't write against a read only slave.
# 从虚拟机3
set c c
# result : (error) READONLY You can't write against a read only slave.
```

#### 5. 成功redis-sentinel日志

```
# 查看日志：
tailf /var/log/redis/sentinel.log
```

成功日志，+slave slave包含两台从库的地址，+sentinel sentinel包含两台哨兵的id

```
57611:X 21 Oct 02:03:27.777 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
57611:X 21 Oct 02:03:27.777 # Sentinel ID is 42975048e2f70d0f4d718f77427930c16bc0b522
57611:X 21 Oct 02:03:27.777 # +monitor master mymaster 172.16.48.129 6379 quorum 2
57611:X 21 Oct 02:03:27.778 * +slave slave 172.16.48.131:6379 172.16.48.131 6379 @ mymaster 172.16.48.129 6379
57611:X 21 Oct 02:03:27.779 * +slave slave 172.16.48.130:6379 172.16.48.130 6379 @ mymaster 172.16.48.129 6379
57611:X 21 Oct 02:03:29.767 * +sentinel sentinel 29222b827e3739b564939c6f20eb610802b48706 172.16.48.130 26379 @ mymaster 172.16.48.129 6379
57611:X 21 Oct 02:03:29.769 * +sentinel sentinel ea3c41804d2840a4393bbdaf0f32dab321267a9c 172.16.48.131 26379 @ mymaster 172.16.48.129 6379
```

#### 6. 成功sentinel的连接状态

```
# 主虚拟机1
redis-cli -h 172.16.48.129 -p 26379 INFO Sentinel
# result：
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.16.48.129:6379,slaves=2,sentinels=
# 从虚拟机2
redis-cli -h 172.16.48.130 -p 26379 INFO Sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.16.48.129:6379,slaves=2,sentinels=3
# 从虚拟机3
redis-cli -h 172.16.48.131 -p 26379 INFO Sentinel
# Sentinel
sentinel_masters:1
sentinel_tilt:0
sentinel_running_scripts:0
sentinel_scripts_queue_length:0
sentinel_simulate_failure_flags:0
master0:name=mymaster,status=ok,address=172.16.48.129:6379,slaves=2,sentinels=3
```

#### 7. 高可用测试case

哨兵作为对redis实例的监控，通过选举算法保证哨兵的鲁棒性和高可用，所以哨兵至少要部署3台，符合半数原则，需要5或者，7，超过一半，不包含一半存活的时候，才能够选举出leader，才能进行主从的切换功能。
redis服务，至少需要存活一台，才能保证服务正常运行sentinel 选择新 master 的原则是最近可用 且 数据最新 且 优先级最高 且 活跃最久 ！
哨兵高可用测试：分别连接对应的redis服务端，手动停止哨兵，停止主reids服务，看主从是否切换成功。
三哨兵情况：redis实例挂掉两台，剩下一台能够成为主，自动切换

```
# 保持三个哨兵进程都存在的情况下
# 1. 三个终端分别连接redis，使用info replication查看当前连接状态：
# 主虚拟机1
redis-cli -h 172.16.48.129 -p 6379 -a 123456789 info replication
# 从虚拟机2
redis-cli -h 172.16.48.130 -p 6379 -a 123456789 info replication
# 从虚拟机3
redis-cli -h 172.16.48.131 -p 6379 -a 123456789 info replication

# 2. 停止当前role:master的对应的redis服务,重新检查状态看是否切换
systemctl stop redis
# 虚拟机1的实例转换成为主的redis

# 3. 继续停止剩下两台：role：master的对应的redis服务，重新检查连接状态看是否切换
systemctl stop redis
# 虚拟机3的实例转换成为了主的redis
# 切换顺利，实现高可用
```

两哨兵情况：redis实例挂掉两台，剩下一台能够成为主，自动切换

```
# 将全部虚拟机的redis + sentinel重新启动
systemctl start redis
systemctl start redis-sentinel
# 停止虚拟机1的redis-sentinel，重新执行哨兵的案例测试
systemctl stop redis-sentinel
# 分别停止对应master实例的redis，最终剩下一台实例，成为了master，能够自动切换
```

一哨兵情况：redis实例无法主从切换

```
# 将全部虚拟机的redis + sentinel重新启动
systemctl start redis
systemctl start redis-sentinel
# 停止虚拟机1和2d的redis-sentinel，重新执行哨兵的案例测试
systemctl stop redis-sentinel
# 分别停止对应master实例的redis，最终剩下一台实例，无法实现主从切换
```

#### 8. web服务连接测试

创建一个web项目，使用项目进行服务器连接验证，暂时不提供。使用spring boot + spring-data-redis进行测试

### 基本运维操作

#### 基本命令操作

```
# 启动
systemctl start redis
systemctl start redis-sentinel
# 重启
systemctl restart redis
systemctl restart redis-sentinel
# 停止
systemctl stop redis
systemctl stop redis-sentinel
# 开机启动
systemctl enable redis
systemctl enable redis-sentinel
# 关闭开机启动
systemctl disable redis
systemctl disable redis-sentinel
# 卸载,停止redis服务，sentinel服务之后，关闭开机启动，进行卸载
yum remove redis -y
```

#### 配置文件目录

redis配置文件：/etc/redis.conf

redis-sentinel配置文件：/etc/redis-sentinel.conf

#### 日志文件目录

redis日志：/var/log/redis/redis.log

redis-sentinel日志：/var/log/redis/sentinel.log

#### 持久化文件目录

redis-dump文件：/var/lib/redis/dump.rdb

redis-appendonly文件：/var/lib/redis/appendonly.aof

### 参考教程

[在Centos中yum安装和卸载软件的使用方法](http://blog.51cto.com/lxsym/297156)

[Markdown插入表格语法](https://www.jianshu.com/p/2df05f279331)

[配置sentinel down-after-milliseconds mymaster 5000报错，解决方案](https://blog.csdn.net/shenlf_bk/article/details/80702463)

[Redis 高可用之 Sentinel 部署与原理](http://zhaif.us/2016/01/12/redis-2/)