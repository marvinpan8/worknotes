# redis7单机安装

```properties
cd /usr/src/
tar -zxvf redis-7.0.15.tar.gz

yum -y install gcc

cd /usr/src/redis-7.0.15
make && make install PREFIX=/usr/local/redis7

mkdir /data/redis6379/{conf,data,log} -p

cd /data/redis6379/conf
vim redis.conf
```


```properties
port 6379
daemonize yes

pidfile "/data/redis6379/data/redis.pid"
loglevel notice
logfile "/data/redis6379/log/redis.log"
databases 16
#save 1800 1
dbfilename "dump.rdb"
dir "/data/redis6379data"

maxmemory 1gb
maxmemory-policy volatile-lru

appendonly no
appendfilename "appendonly.aof"
slowlog-log-slower-than 10000
slowlog-max-len 128

requirepass "admin"
```

vim /etc/profile

```properties
export PATH="/usr/local/redis7/bin:$PATH"
```

source /etc/profile



 vim /etc/systemd/system/redis.service  

```properties
[Unit]
Description=Redis 6379 Server 
After=network.target  

[Service]
Type=forking 
ExecStart=/usr/local/redis7/bin/redis-server /data/redis6379/conf/redis.conf  
ExecStop=/usr/local/redis7/bin/redis-cli -p 6379 shutdown 
Restart=on-failure 
User=root 

[Install]
WantedBy=multi-user.target  
```

```properties
systemctl daemon-reload  && systemctl enable redis && systemctl start redis
ps -ef | grep redis
```

