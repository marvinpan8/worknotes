### 起因

有一天生产环境的API发现大部分都无法链接，查日志发现是redis报错，主要的错误信息为 `ERR max number of clients reached` 通过命令`info clients`  查询，发现连接数超高

```
# Clients
connected_clients:9793
client_recent_max_input_buffer:2
client_recent_max_output_buffer:0
blocked_clients:0
```

由于redis默认连接数最大值为10000，导致无法连接redis而出现错误。
之后用 client list  导出结果，发现链接的是 900+，属于正常的连接数。

使用 `config get timeout  命令查看连接超时时间

```
1) "timeout"
2) "0"
```

在redis的官方文档有介绍
Close the connection after a client is idle for N seconds (0 to disable)
当设置为0时，超时的连接在N秒后会释放。（0关闭该功能）

### 检查集群

```
cluster info
```

集群状态正常。
目前线上的环境是 3主3从

114 连接数一直维持在 9400多
115 和 116 链接数在 800 和 900 左右

### 疑问

本想导出所有连接，查看是那台机器的占用的链接数多

```bash
CLIENT LIST #获取客户端列表
CLIENT SETNAME #设置当前连接点redis的名称
CLIENT GETNAME #查看当前连接的名称
CLIENT KILL ip:port #杀死指定连接
```

通过 client list  可以直接在 控制台查看，但是内容太多不好分辨。
试用命令： "client list" > list.txt  不能导出，用程序导出，能看到连接数只有 900 多个。最后确认活动链接数为 900 多个。

### 解决

#### 第一步

发现114机器的 Redis server_log 日志为  4.38G,停集群，删除日志

#### 第二步

设置过期时间，为了释放 idle  态的链接

将链接设置为 300 s .

用命令设置：
`CONFIG SET timeout 30  这种方式下次启动会失效。
在设置了时间之后，redis连接数恢复了正常。

#### 第三步

请求 API 压测时，发现链接增长到900，5 分钟后链接到 126 。初步判断运行结果正常。

### 参考资料

<http://www.redis.cn/commands/client-list.html>
<https://www.jianshu.com/p/70f3b68a7fd7>