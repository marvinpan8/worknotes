## 一、AOF导入方式

1. 源实例生成aof数据

   ```bash
   # 清空上文目标实例全部数据
   [root@172.20.0.1 ~]# redis-cli -h 10.9.1.16 -p 6379 -a invest0755 flushall
   OK
   # 源实例开启aof功能，将在dir目录下生成appendonly.aof文件
   [root@172.20.0.1 ~]# redis-cli -h 10.9.1.16 -p 6380 config set appendonly yes
   OK
   ```

2. 目标实例导入aof数据
   ```bash
   # 假设appendonly.aof就在当前路径下
   [root@172.20.0.1 ~]# redis-cli -h 10.9.1.16 -p 6379 -a invest0755 --pipe < appendonly.aof
   All data transferred. Waiting for the last reply...
   Last reply received from server.
   errors: 0, replies: 5
   # 源实例关闭aof功能
   [root@172.20.0.1 ~]# redis-cli -h 10.9.1.16 -p 6380 config set appendonly no
   OK
   ```

## 二、redis-dump方式

1.安装redis-dump工具

```bash
`[root@172.20.0.3 ~]``# yum install ruby rubygems ruby-devel -y``# 更改gem源``[root@172.20.0.3 ~]``# gem sources -a http://ruby.taobao.org``Error fetching http:``//ruby``.taobao.org:``    ``bad response Not Found 404 (http:``//ruby``.taobao.org``/specs``.4.8.gz)`
```

　　访问http://ruby.taobao.org，公告通知镜像维护站点已迁往Ruby China镜像

```bash
`#gem sources --add https://gems.ruby-china.org/ --remove https://rubygems.org/``[root@172.20.0.3 ~]``# gem sources --add http://gems.ruby-china.org/ --remove http://rubygems.org/``http:``//gems``.ruby-china.org/ added to sources``source` `http:``//rubygems``.org/ not present ``in` `cache``[root@172.20.0.3 ~]``# gem sources -l``*** CURRENT SOURCES ***` `http:``//gems``.ruby-china.org/``[root@172.20.0.3 ~]``# gem install redis-dump -V`
```

　　2.redis-dump导出

```
`[root@172.20.0.3 ~]``# redis-dump -u :password@172.20.0.1:6379 > 172.20.0.1.json`
```

　　3.redis-load导入

```
`[root@172.20.0.3 ~]``# cat 172.20.0.1.json | redis-load -u :password@172.20.0.2:6379`
```

## 三、源实例db0迁移至目标实例db1

```bash
[root@172.20.0.1 ~]``# cat redis_mv.sh
#!/bin/bash
redis-cli -h 172.20.0.1 -p 6379 -a password -n 0 keys "*" | while read key
do
    redis-cli -h 172.20.0.1 -p 6379 -a password -n 0 --raw dump $key | perl -pe 'chomp if eof' | redis-cli -h 172.20.0.2 -p 6379 -a password -n 1 -x restore $key 0
    echo "migrate key $key"
done
```