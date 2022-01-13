# Clickhouse 安装与部署

## 环境要求
Clickhouse 仅支持Linux 且必须支持SSE4.2 指令集, 这里用Centos7进行演示

```bash
grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
```

得出下列结果

得出下列结果

```bash
SSE 4.2 supported
```


如果服务器不支持SSE指令集，则不能直接下载预编译安装包，需要通过源码编译特定版本进行安装

```bash
systemctl stop firewalld.service 
systemctl disable firewalld.service 
```

关闭防火墙以及防火墙自启动

## 版本选择及下载
```bash
mkdir -p /jrtz/clickhouse/
```


先创建目录存放下载的文件,  然后选择所需要的的版本进行下载

https://packagecloud.io/Altinity/clickhouse

选择所需的安装包，只需要下面四种即可，其中el/7表示为centos7版本(此处为示例版本)：包名

```bash
clickhouse-server-common-20.8.3.18-1.el7.x86_64.rpm
clickhouse-server-20.8.3.18-1.el7.x86_64.rpm el/7
clickhouse-common-static-20.8.3.18-1.el7.x86_64.rpm
clickhouse-client-20.8.3.18-1.el7.x86_64.rpm
```

## 安装

下载上述4类安装包后到对应目录中，进行安装

```bash
rpm -ivh ./*.rpm
```

``` bash
[root@vmsrv-010-214 clickhouse]# rpm -ivh ./*.rpm
Preparing...                          ################################# [100%]
Updating / installing...
   1:clickhouse-server-common-20.8.3.1################################# [ 25%]
   2:clickhouse-common-static-20.8.3.1################################# [ 50%]
   3:clickhouse-server-20.8.3.18-1.el7################################# [ 75%]
Create user clickhouse.clickhouse with datadir /var/lib/clickhouse
   4:clickhouse-client-20.8.3.18-1.el7################################# [100%]
Create user clickhouse.clickhouse with datadir /var/lib/clickhouse
```

## Clickhouse目录结构

1. **/etc/clickhouse-server** : 服务端的配置文件目录，包括全局配置config.xml 和用户配置users.xml，其中如需要 **外网访问** 则需要打开config.xml中更改配置,  需要放开`<listen_host>::</listen_host>`的注释即可

   `vim /etc/clickhouse-server/config.xml`

   ```xml
   <interserver_http_host>example.yandex.ru</interserver_http_host>
       -->
   
       <!-- Listen specified host. use :: (wildcard IPv6 address), if you want to accept connections both with IPv4 and IPv6 from everywhere. -->
       <listen_host>::</listen_host>
       <!-- Same for hosts with disabled ipv6: -->
       <!-- <listen_host>0.0.0.0</listen_host> -->
   
       <!-- Default values - try listen localhost on ipv4 and ipv6: -->
       <!--
       <listen_host>::1</listen_host>
       <listen_host>127.0.0.1</listen_host>
       -->
       <!-- Don't exit if ipv6 or ipv4 unavailable, but listen_host with this protocol specified -->
       <!-- <listen_try>0</listen_try> -->
   ```

2. **/var/lib/clickhouse** : 默认的**数据存储目录**，通常会修改，将数据保存到大容量磁盘路径中，修改config.xml的所有  /var/lib/clickhouse 路径为指定路径，重启即可
3. **/var/log/cilckhouse-server** : 默认保存日志的目录，通常会修改，将数据保存到大容量磁盘路径中

## 启动服务

```bash
$ service clickhouse-server start
Start clickhouse-server service: Path to data directory in /etc/clickhouse-server/config.xml: /var/lib/clickhouse/
DONE
```

可以在/var/log/clickhouse-server/目录中查看日志。

如果服务没有启动，请检查配置文件 /etc/clickhouse-server/config.xml。

你也可以在控制台中直接启动服务：

```
clickhouse-server --config-file=/etc/clickhouse-server/config.xml
```

在这种情况下，日志将被打印到控制台中，这在开发过程中很方便。
如果配置文件在当前目录中，你可以不指定’–config-file’参数。它默认使用’./config.xml’。

你可以使用命令行客户端连接到服务：

```bash
$ clickhouse-client
ClickHouse client version 20.3.12.112.
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 20.3.12 revision 54433.

wbl.clickhouse :) 
```

验证sql

```bash
wbl.clickhouse :) select 1 

SELECT 1

┌─1─┐
│ 1 │
└───┘

1 rows in set. Elapsed: 0.003 sec. 
```

## 参考资料

- [Clickhouse中文文档](https://clickhouse.tech/docs/zh/) 引用日期2020-06-29
- ClickHouse原理解析与应用实践 ．朱凯[引用日期2020-06-29]

---
原文链接：https://blog.csdn.net/wbl381/article/details/106995351/