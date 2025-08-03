# 开源版EMQX-5.8安装

## 环境准备

```properties
localectl set-locale LANG=zh_CN.utf8
source /etc/locale.conf
locale
```

永久修改主机名  vim /etc/hostname 

临时修改主机名

```properties
hostnamectl set-hostname  emqx@172.17.0.30
hostnamectl set-hostname  emqx@172.17.0.31
hostnamectl set-hostname  emqx@172.17.0.32
```

vim /etc/hosts

```properties
172.17.0.30  emqx@172.17.0.30
172.17.0.31  emqx@172.17.0.31
172.17.0.32  emqx@172.17.0.32
```

```properties
mkdir -p /data/emqx/{data,logs}
chown emqx:emqx /data/emqx/data
chown emqx:emqx /data/emqx/logs
```
# 系统调优

## 1. 关闭swap交换分区

Linux 交换分区可能会导致 Erlang 虚拟机出现不确定的内存延迟，严重影响系统的稳定性。 建议永久关闭交换分区。

- 要立即关闭交换分区，执行命令 `sudo swapoff -a`。
- 要永久关闭交换分区，在 `/etc/fstab` 文件中注释掉 `swap` 行，然后重新启动主机。

## 2、Linux 操作系统参数

系统全局允许分配的最大文件句柄数:

```properties
# 2 millions system-wide
sysctl -w fs.file-max=2097152
sysctl -w fs.nr_open=2097152
echo 2097152 > /proc/sys/fs/nr_open
```

允许当前会话 / 进程打开文件句柄数:

```properties
ulimit -n 1048576
```

### /etc/sysctl.conf

持久化 `fs.file-max` 设置到 /etc/sysctl.conf 文件:

vim  /etc/sysctl.conf

```properties
fs.file-max = 1048576
```

sysctl -p  /etc/sysctl.conf

设置服务最大文件句柄数:

vim  /etc/systemd/system.conf  

```properties
DefaultLimitNOFILE=1048576
```

持久化设置允许用户 / 进程打开文件句柄数:

vim /etc/security/limits.conf 

```properties
*      soft   nofile      1048576
*      hard   nofile      1048576
```

## TCP 协议栈网络参数

并发连接 backlog 设置:

```properties
sysctl -w net.core.somaxconn=32768
sysctl -w net.ipv4.tcp_max_syn_backlog=16384
sysctl -w net.core.netdev_max_backlog=16384
```

可用知名端口范围:

```properties
sysctl -w net.ipv4.ip_local_port_range='1024 65535'
```

TCP Socket 读写 Buffer 设置:

```properties
sysctl -w net.core.rmem_default=262144
sysctl -w net.core.wmem_default=262144
sysctl -w net.core.rmem_max=16777216
sysctl -w net.core.wmem_max=16777216
sysctl -w net.core.optmem_max=16777216
    
#sysctl -w net.ipv4.tcp_mem='16777216 16777216 16777216'
sysctl -w net.ipv4.tcp_rmem='1024 4096 16777216'
sysctl -w net.ipv4.tcp_wmem='1024 4096 16777216'
```

TCP 连接追踪设置:

```properties
sysctl -w net.nf_conntrack_max=1000000
sysctl -w net.netfilter.nf_conntrack_max=1000000
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
```

TIME-WAIT Socket 最大数量、回收与重用设置:

```properties
sysctl -w net.ipv4.tcp_max_tw_buckets=1048576
    
# 注意：不建议开启該设置，NAT 模式下可能引起连接 RST
# sysctl -w net.ipv4.tcp_tw_recycle=1
# sysctl -w net.ipv4.tcp_tw_reuse=1
```

FIN-WAIT-2 Socket 超时设置:

```properties
sysctl -w net.ipv4.tcp_fin_timeout=15
```

vim /etc/sysctl.d/kubernetes.conf

```properties
net.ipv4.tcp_max_tw_buckets=1048576
net.ipv4.tcp_max_syn_backlog=16384
net.netfilter.nf_conntrack_max=1000000
```

sysctl -p /etc/sysctl.d/kubernetes.conf



## Erlang 虚拟机参数

优化设置 Erlang 虚拟机启动参数，

vim /etc/emqx/emqx.conf

```properties
## 设置 Erlang 系统同时存在的最大端口数
node.max_ports = 2097152
```



## EMQX 消息服务器参数

https://docs.emqx.com/zh/emqx/v5.8/

### 监听器 Acceptor 参数

为了优化连接处理能力，可以通过修改 `etc/emqx.conf` 配置文件，调整监听器 acceptor 池大小和 `max_connections` 限制。

以 TCP 监听器为例：

vim /etc/emqx/emqx.conf

```properties
## TCP 监听器配置
listeners.tcp.$name.acceptors = 64
listeners.tcp.$name.max_connections = 1024000
```

- `acceptors`：用于处理入站连接的 acceptor 进程数量。
- `max_connections`：允许的最大并发连接数。




### 下载链接开源版
https://www.emqx.com/zh/downloads/broker/

选   v5.8.7/emqx-5.8.7-el7-amd64.rpm

## RPM安装

```bash
yum install emqx-5.8.7-el7-amd64.rpm -y
```

vim /usr/lib/systemd/system/emqx.service

```
ExecStart=/bin/bash /usr/bin/emqx foreground
```



vim /etc/emqx/emqx.conf

```properties
node {
  name = "emqx@172.17.0.32"
  data_dir = "/data/emqx/data"
  max_ports = 2097152
}

cluster {
  name = emqxcl
  discovery_strategy = static
  static {							
     seeds = ["emqx@172.17.0.30", "emqx@172.17.0.31", "emqx@172.17.0.32"]  
  }									
}

log {
  file_handlers.default {
    level = warning
    file = "/data/emqx/logs/emqx.log"
  }
}
```

## 启动

```properties
systemctl daemon-reload  && systemctl enable emqx && systemctl start emqx
```

## 查看集群和日志

```properties
emqx ctl cluster status
journalctl -u emqx -f -n 500
systemctl status emqx
tail -f -n 500 /data/emqx/logs/emqx.log.1
```

---



#  登陆界面

- 浏览器输入：`http://172.17.0.30:18083`
- 默认账号 / 密码：`admin/public`

## 端口占用

EMQX 默认使用以下端口，请确保这些端口未被其他应用程序占用，并按照需求开放防火墙以保证 EMQX 正常运行。

| 端口      | 协议 | 描述                                                         |
| --------- | ---- | ------------------------------------------------------------ |
| **1883**  | TCP  | MQTT over TCP 监听器端口，主要用于未加密的 MQTT 连接。       |
| **8883**  | TCP  | MQTT over SSL/TLS 监听器端口，用于加密的 MQTT 连接。         |
| **8083**  | TCP  | MQTT over WebSocket 监听器端口，使 MQTT 能通过 WebSocket 进行通信。 |
| **8084**  | TCP  | MQTT over WSS (WebSocket over SSL) 监听器端口，提供加密的 WebSocket 连接。 |
| **18083** | HTTP | EMQX Dashboard 和 REST API 端口，用于管理控制台和 API 接口。 |
| 4370      | TCP  | Erlang 分布式传输端口，根据节点名称不同实际端口可能是 `BasePort (4370) + Offset`。 |
| 5370      | TCP  | 集群 RPC 端口（在 Docker 环境下为 5369），根据节点名称不同实际端口可能是 `BasePort (5370) + Offset`。 |

> 提示
>
> 即使没有组建集群，EMQX 也会监听 4370 跟 5370 端口。这 2 个端口固定无法修改，且会根据节点名称（`Name@Host`）中 Name 部分的数字后缀决定 Offset，没有数字后缀则默认为 0。更多信息请参考[集群内通信端口](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/security.html#%E9%9B%86%E7%BE%A4%E5%86%85%E9%80%9A%E4%BF%A1%E7%AB%AF%E5%8F%A3)。



## 文件和目录

EMQX 安装完成后会创建一些目录用来存放运行文件和配置文件，存储数据以及记录日志。

不同安装方式得到的文件和目录位置有所不同，具体如下:

| 目录       | 描述              | 压缩包解压安装 | 二进制包安装             |
| ---------- | ----------------- | -------------- | ------------------------ |
| `etc`      | 静态配置文件      | `./etc`        | `/etc/emqx`              |
| `data`     | 数据和配置文件    | `./data`       | `/var/lib/emqx`          |
| `log`      | 日志文件          | `./log`        | `/var/log/emqx`          |
| `releases` | 启动相关的脚本    | `./releases`   | `/usr/lib/emqx/releases` |
| `bin`      | 可执行文件        | `./bin`        | `/usr/lib/emqx/bin`      |
| `lib`      | Erlang 代码       | `./lib`        | `/usr/lib/emqx/lib`      |
| `erts-*`   | Erlang 虚拟机文件 | `./erts-*`     | `/usr/lib/emqx/erts-*`   |
| `plugins`  | 插件              | `./plugins`    | `/usr/lib/emqx/plugins`  |

> TIP
>
> 1. 压缩包解压安装时，目录相对于软件所在目录；
> 2. Docker 容器使用压缩包解压安装的方式，软件安装于 `/opt/emqx` 目录中；
> 3. `data`、`log`、`plugins` 目录可以通过配置文件设置，建议将 `data` 目录挂载至高性能磁盘以获得更好的性能。但对于属于同一集群的节点， `data` 目录的配置应该相同。更多关于集群的介绍，见[集群章节](https://docs.emqx.com/zh/emqx/latest/deploy/cluster/introduction.html)。

接下来我们将详细介绍下其中的部分目录，其中包含的文件和子文件夹。

| 目录 | 描述                 | 权限 | 目录文件                                                     |
| ---- | -------------------- | ---- | ------------------------------------------------------------ |
| bin  | 存放可执行文件       | 读   | `emqx` 和`emqx.cmd`：EMQX 的可执行文件，具体使用可以查看[命令行接口](https://docs.emqx.com/zh/emqx/latest/admin/cli.html)。 |
| etc  | 存放配置文件         | 读   | `emqx.conf`：EMQX 的主配置文件，默认包含常用的配置项。  `emqx-example-en.conf`：EMQX 示例配置文件，包含所有可选的配置项。  `acl.conf`：默认 ACL 规则。  `vm.args`：Erlang 虚拟机的运行参数。  `certs/`：X.509 的密钥和证书文件。这些文件被用于 EMQX 的 SSL/TLS 监听器；当要与和外部系统集成时，也可用于建立 SSL/TLS 连接。 |
| data | 存放 EMQX 的运行数据 | 写   | `authz`：Dashboard 或 REST API 上传的 [基于文件进行授权](https://docs.emqx.com/zh/emqx/latest/access-control/authz/file.html) 规则内容。  `certs`：Dashboard 或 REST API 上传的证书。  `configs`：启动时生成的配置文件，或者从 Dashboard/REST API/CLI 进行功能设置时覆盖的配置文件。  `mnesia`：内置数据库目录，用于存储自身运行数据，例如告警记录、客户端认证与权限数据、Dashboard 用户信息等数据，**一旦删除该目录，所有业务数据将丢失。**  — 可包含以节点命名的子目录，如 `emqx@127.0.0.1`；如节点被重新命名，应手动将旧的目录删除或移走。  — 可通过 `emqx_ctl mnesia` 命令查询 EMQX 中 Mnesia 数据库的系统信息，具体请查看 [管理命令 CLI](https://docs.emqx.com/zh/emqx/latest/admin/cli.html)。  `patches`：用于存储热补丁 `.beam` 文件，用于补丁修复。  `trace`: 在线日志追踪文件目录。   在生产环境中，建议定期备份该文件夹下除 `trace` 之外的所有目录。 |
| log  | 日志文件             | 读   | `emqx.log.*`：EMQX 运行时产生的日志文件，具体请查看[日志](https://docs.emqx.com/zh/emqx/latest/observability/log.html)。 |

> TIP
>
> EMQX 的配置项存储在 `etc` 和 `data/configs` 目录下，二者的主要区别是 `etc` 目录存储**只读**的配置文件，用户通过 Dashboard 和 REST API 提交的配置将被保存到 `data/configs` 目录下，并支持在运行时进行热更新。
>
> - `etc/emqx.conf`
> - `data/configs/cluster.hocon`
>
> EMQX 读取这些配置并将其合并转化为 Erlang 原生配置文件格式，以便在运行时应用这些配置。
>

------



#  部署 nginx

```properties
cat << EOF | tee /k8s/nginx/conf/kube-nginx.conf
    worker_processes auto;
    error_log /var/log/kube-nginx-error.log info;

    events {
        multi_accept on;
        use epoll;
        worker_connections  1024;
    }

    stream {
        upstream emqx_tcp {
            hash $remote_addr consistent;
            server 172.17.0.30:1883 max_fails=3 fail_timeout=30s;
            server 172.17.0.31:1883 max_fails=3 fail_timeout=30s;
            server 172.17.0.32:1883 max_fails=3 fail_timeout=30s;
        }
        
        upstream emqx_ws {
            hash $remote_addr consistent;
            server 172.17.0.30:8083 max_fails=3 fail_timeout=30s;
            server 172.17.0.31:8083 max_fails=3 fail_timeout=30s;
            server 172.17.0.32:8083 max_fails=3 fail_timeout=30s;
        }
        
        upstream emqx_http {
            hash $remote_addr consistent;
            server 172.17.0.30:18083 max_fails=3 fail_timeout=30s;
            server 172.17.0.31:18083 max_fails=3 fail_timeout=30s;
            server 172.17.0.32:18083 max_fails=3 fail_timeout=30s;
        }
        

        server {
            listen 11883;
            proxy_connect_timeout 2s;
            proxy_timeout 120;
            proxy_pass emqx_tcp;
        }
        server {
            listen 28083;
            proxy_connect_timeout 2s;
            proxy_timeout 120;
            proxy_pass emqx_ws;
        }
        server {
            listen 38083;
            proxy_connect_timeout 2s;
            proxy_timeout 60;
            proxy_pass emqx_http;
        }
    }
EOF
```
#### 配置 systemd unit 文件，启动服务

```properties
cat << EOF | tee /etc/systemd/system/kube-nginx.service
[Unit]
Description=emqx nginx proxy
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
ExecStartPre=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx -t
ExecStart=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx
ExecStop=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx -s quit
ExecReload=/k8s/nginx/sbin/kube-nginx -c /k8s/nginx/conf/kube-nginx.conf -p /k8s/nginx -s reload
PrivateTmp=true
Restart=always
RestartSec=5
StartLimitInterval=0
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
#### 启动 kube-nginx 服务：

```properties
systemctl daemon-reload && systemctl enable kube-nginx && systemctl start kube-nginx
```

#### 检查 kube-nginx 服务运行状态

```properties
systemctl status kube-nginx |grep 'Active:'
netstat -anp |grep 38083
netstat -anp |grep 28083
netstat -anp |grep 11883
```

确保状态为 `active (running)`，否则到 master 节点查看日志，排查原因：

```properties
journalctl -f -u kube-nginx
```
------

#  部署 keepalived

### 入口3节点安装

```properties
yum install -y gcc openssl-devel popt-devel
mkdir -p /k8s/keepalived && cd /k8s/keepalived
wget http://www.keepalived.org/software/keepalived-2.3.4.tar.gz
tar -zxvf keepalived-2.3.4.tar.gz && rm -f keepalived-2.3.4.tar.gz
cd keepalived-2.3.4 && mkdir keepalived-prefix
# --prefix：指定安装路径
./configure --prefix=$(pwd)/keepalived-prefix
# 安装
make && make install
#复制文件
cp keepalived/etc/init.d/keepalived /etc/init.d
cp keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cd keepalived-prefix/ && mkdir /etc/keepalived
#复制文件
cp etc/keepalived/keepalived.conf.sample /etc/keepalived/
```

### 【30】节点

```properties
cat << EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     pantianjun@changxing28.com
   }
   notification_email_from keepalived@changxing28.com
   smtp_server 172.17.0.110
   smtp_connect_timeout 30
   router_id emqx@172.17.0.30
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 99
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.99/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```

### 【31-32】节点

```properties
cat << EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     pantianjun@changxing28.com
   }
   notification_email_from keepalived@changxing28.com
   smtp_server 172.17.0.110
   smtp_connect_timeout 30
   router_id emqx@172.17.0.32
}

vrrp_script chk_http_port {
    script "/etc/keepalived/check_nginx.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    virtual_router_id 99
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass cxkj0755
    }
    virtual_ipaddress {
        172.17.0.99/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```

#### 验证是否安装成功
```properties
systemctl daemon-reload && systemctl enable keepalived && systemctl start keepalived
```

#### 查看日志文件
```properties
tail -f -n 500 /var/log/messages
```

------

## emqx对外IP端口

```properties
emqx tcp：  172.17.0.66:11883
emqx ws：  172.17.0.66:28083
emqx dashboard：  172.17.0.66:38083
```
