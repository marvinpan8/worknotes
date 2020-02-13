### redis-5.0.4配置

- 修改内核参数

```bash
vim /proc/sys/net/core/somaxconn
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
# 最大内存物理内存/4，本例：16G=17179869184 byte
maxmemory 17179869184
#内存策略，不驱逐
maxmemory-policy noeviction
```

