### redis-5.0.4配置

- 修改内核参数

```bash
cat /proc/sys/net/core/somaxconn
echo 511 > /proc/sys/net/core/somaxconn
#在/etc/sysctl.conf中添加如下
net.core.somaxconn=511
#然后在终端中执行
sysctl -p
```



```bash
bind 192.168.10.230
# Close the connection after a client is idle for N seconds (0 to disable)
timeout 300
logfile "/jrtz/redis/redis-5.0.4/log/redis6379.log"
databases 256
# 先创建文件夹 mkdir data
dir ./data/
requirepass zaq1xsw2
# 最大内存物理内存的3/4，本例：16G=17179869184 byte(B) * 3/4 = 12884901888
maxmemory 12884901888
#内存策略，不驱逐
maxmemory-policy noeviction
```

### 慢日志查询

```bash
slowlog get 100
# 长度
SLOWLOG LEN
# 清空
SLOWLOG RESET
```
```bash
127.0.0.1:6379> SLOWLOG GET 3

1) 1) (integer) 14                # 唯一性(unique)的日志标识符
   2) (integer) 1522808219        # 被记录命令的执行时间点，以 UNIX 时间戳格式表示
   3) (integer) 16                # 查询执行时间，以微秒为单位
   4) 1) "keys"                   # 执行的命令，以数组的形式排列
      2) "*"                      # 这里完整的命令是 "keys *"
   5) "192.168.10.224:62918"      # 客户端IP和端口
```



### 开机自启动

vim /etc/systemd/system/redis.service 

```toml
[Unit]
Description=redis
After=network.target

[Service]
Type=forking
PIDFile=/var/run/redis6379.pid
ExecStart=/jrtz/redis-6.0.4/src/redis-server /jrtz/redis-6.0.4/redis.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

---

## 问题记录

### 1. [Redis连接数居高不下的问题](https://www.cnblogs.com/qingyanxiaochen/p/11091439.html)

有一天生产环境的API发现大部分都无法链接，查日志发现是redis报错，主要的错误信息为 

```
ERR max number of clients reached
```

通过命令`info clients`  查询，发现连接数超高

```bash
# Clients
connected_clients:9793
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0
```

由于redis默认连接数最大值为10000，导致无法连接redis而出现错误。
之后用 client list  导出结果，发现链接的是 900+，属于正常的连接数。

使用 `config get timeout`  命令查看连接超时时间

```bash
1) "timeout"
2) "0"
```

使用 `client list`  命令查看客户端列表，查看具体连接数多的DB

