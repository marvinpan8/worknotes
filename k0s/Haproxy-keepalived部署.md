# haproxy部署

**Haproxy 是七层 http 代理**

- 官网：https://docs.haproxy.org/
- 下载：https://src.fedoraproject.org/repo/pkgs/haproxy/  
- 最新版本 3.4.2，ubuntu：2.8.16-0ubuntu0.24.04.3

```properties
sudo apt update && sudo apt install -y haproxy
cat /usr/lib/systemd/system/haproxy.service
cat /etc/haproxy/haproxy.cfg
```

### haproxy.service

- ubuntu 自带的 service

```properties
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
Documentation=file:/usr/share/doc/haproxy/configuration.txt.gz
After=network-online.target rsyslog.service
Wants=network-online.target

[Service]
# ubuntu
EnvironmentFile=-/etc/default/haproxy
# centos
EnvironmentFile=-/etc/sysconfig/haproxy
BindReadOnlyPaths=/dev/log:/var/lib/haproxy/dev/log
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "PIDFILE=/run/haproxy.pid" "EXTRAOPTS=-S /run/haproxy-master.sock"
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -p $PIDFILE $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -Ws -f $CONFIG -c -q $EXTRAOPTS
# 信号触发的优雅重载，而非暴力终止
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=always
SuccessExitStatus=143
Type=notify
```

### 配置 rsyslog

```properties
# vim /etc/rsyslog.conf  确保以下两行未被注释
module(load="imudp")
input(type="imudp" port="514")

# 重启 rsyslog
suso systemctl restart rsyslog
```

### 配置 haproxy.cfg

- https://docs.haproxy.org/2.8/configuration.html

#### sudo vim /etc/haproxy/haproxy.cfg

- 修改 IP

```properties
sudo tee /etc/haproxy/haproxy.cfg << EOF
global
    # 使用 UDP 日志（兼容 chroot）
	log 127.0.0.1 local0 info
	# 将进程的根目录限制在 /var/lib/haproxy 目录下
	chroot /var/lib/haproxy
	# 最大连接数
	maxconn 10000
	# 允许脚本交互
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

defaults
	log	global
	# 使用 HTTP 模式
	mode http
    # HTTP 详细格式
	option httplog
	# 过滤空连接，健康检查和探测流量等
	option dontlognull

	timeout connect 10s
	# 1. 增大超时时间（适应大文件传输）
	timeout client 120s
	timeout server 120s
	timeout check 5s

# ========== 第一组服务（S3 / SeaweedFS）==========
frontend s3-frontend
	bind :9333
	# 保持与客户端的连接（Keep-Alive），但在后端响应完成后，主动关闭与后端的连接。
	option http-server-close
	# 添加 X-Forwarded-For 头部, 本机除外
	option forwardfor except 127.0.0.1
	default_backend s3-backend

backend s3-backend
	balance roundrobin
	# 启用 HTTP 复用（提升性能）
	option http-keep-alive
	
	# 简单 TCP 检查：只检测端口是否可连接
    option tcp-check
    tcp-check connect
	
	server seaweed-1 10.10.10.103:8333 check inter 3000 rise 2 fall 3
	server seaweed-2 10.10.10.104:8333 check inter 3000 rise 2 fall 3
	server seaweed-3 10.10.10.105:8333 check inter 3000 rise 2 fall 3
	
# ========== 第二组服务（新增）==========
frontend zot-frontend
    bind :5000
    option http-server-close
    option forwardfor except 127.0.0.1
    default_backend zot-backend

backend zot-backend
    balance roundrobin
    option http-keep-alive
    option tcp-check
    tcp-check connect
    server zot-1 10.10.20.201:5000 check inter 3000 rise 2 fall 3
    server zot-2 10.10.20.202:5000 check inter 3000 rise 2 fall 3
    server zot-3 10.10.20.203:5000 check inter 3000 rise 2 fall 3

# 监控页面
listen admin_stats
    bind :11001
    mode http
    stats refresh 30s
    stats uri /admin
    stats realm HAProxy\ Statistics
    stats auth admin:Ekemp668
    stats show-node
EOF
```
#### 验证是否安装成功
```properties
# 1. 检查语法
haproxy -f /etc/haproxy/haproxy.cfg -c
sudo systemctl reload haproxy
#----重启查看状态--------------------
sudo systemctl restart haproxy
sudo systemctl status haproxy
# 日志
journalctl -xeu haproxy
sudo tail -f -n 500 /var/log/haproxy.log
```

#### 访问页面
```properties
curl http://10.10.10.66:11001/admin
curl -Iv http://10.10.10.104:9333/
curl -Iv http://10.10.10.105:9333/
curl -Iv http://10.10.10.66:9333/
```



---
#  ■■■ 部署 keepalived

```properties
sudo apt update && sudo apt install -y keepalived
cat /usr/lib/systemd/system/keepalived.service
```
## keepalived.service

```properties
[Unit]
Description=Keepalive Daemon (LVS and VRRP)
After=network-online.target
Wants=network-online.target
Documentation=man:keepalived(8)
Documentation=man:keepalived.conf(5)
Documentation=man:genhash(1)
Documentation=https://keepalived.org
ConditionFileNotEmpty=/etc/keepalived/keepalived.conf

[Service]
Type=notify
EnvironmentFile=-/etc/default/keepalived
ExecStart=/usr/sbin/keepalived --dont-fork $DAEMON_ARGS
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
```

## check_haproxy.sh

配置系统日志文件

```properties
sudo tee /etc/rsyslog.d/keepalived.conf << 'EOF'
local0.*    /var/log/keepalived/keepalived.log
local0.*    stop 
EOF

# 创建目录
sudo mkdir -p /var/log/keepalived
# 设置属主（与 rsyslog 用户匹配）
sudo chown syslog:adm /var/log/keepalived
# 设置目录权限
sudo chmod 755 /var/log/keepalived
# 创建日志文件
sudo touch /var/log/keepalived/keepalived.log
# 设置文件权限
sudo chown syslog:adm /var/log/keepalived/keepalived.log
sudo chmod 640 /var/log/keepalived/keepalived.log

# 重启 rsyslog 使配置生效
sudo systemctl restart rsyslog

# 测试日志是否正常写入
logger -p local0.info -t "TEST" "Keepalived test message"
# 查看结果
tail -f /var/log/keepalived/keepalived.log
```



默认每隔3秒钟执行一次检测脚本，检查nginx服务是否启动，如果没启动就把nginx服务启动起来，如果启动不成功，就把keepalived服务down掉，让漂浮到备keepalived上

```properties
sudo tee /etc/keepalived/check_haproxy.sh << 'EOF'
#!/bin/bash

TAG="KEEPALIVED_HAPROXY"

if ! systemctl is-active --quiet haproxy; then
    logger -p local0.err -t "$TAG" "HAProxy is down, attempting restart"
    systemctl restart haproxy
    sleep 5
    if ! systemctl is-active --quiet haproxy; then
        logger -p local0.err -t "$TAG" "HAProxy restart failed, stopping keepalived"
        systemctl stop keepalived
    fi
fi
EOF

# 修正执行权限
sudo chmod +x /etc/keepalived/check_haproxy.sh

# 测试日志是否正常写入
sudo systemctl stop haproxy
sudo /etc/keepalived/check_haproxy.sh
# 查看结果
tail -f -n 500 /var/log/keepalived/keepalived.log
```

> 注意：检测脚本一定要写在vrrp_instance的前面也就是上面，而且花括号一定要有空格，追踪trace_script一定要在vip的后面，多少人栽在了这上面好多小时



## 配置 keepalived.conf

官网配置介绍：https://keepalived.org/documentation/keepalived-conf/?h=#vrrp-scripts

### 【105】节点

- **修改网卡名称(ens192?):  ip a** 
- **修改唯一ID**：virtual_router_id 改成 66
- **修改 lvs_id: S3_API**
- **密码不能超过8个字符**

```properties
sudo tee /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
   notification_email {
     java@ekemp.com.cn
   }
   notification_email_from 2355541806@qq.com
   smtp_server smtp.qq.com
   smtp_connect_timeout 30
   
   router_id S3_API
   max_auto_priority 50
   
   enable_script_security
   script_user root
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    fall 2
    rise 2
    
    user root
}

vrrp_instance VI_1 {
    state MASTER
    interface bond0
    virtual_router_id 66
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp668
    }
    virtual_ipaddress {
        10.10.10.66/24
    }
    
    track_script {                      
        chk_haproxy
    }
}
EOF
```

#### 【104】节点

- **修改网卡名称(ens192?):  ip a** 
- **修改唯一ID：virtual_router_id 改成 66**
- **修改priority：90**
- **修改 lvs_id: S3_API**
- **密码不能超过8个字符**
- **指定脚本执行用户root:   script_user root**

```properties
sudo tee /etc/keepalived/keepalived.conf << EOF
! Configuration File for keepalived

global_defs {
   notification_email {
     java@ekemp.com.cn
   }
   notification_email_from 2355541806@qq.com
   smtp_server smtp.qq.com
   smtp_connect_timeout 30
   
   router_id S3_API
   max_auto_priority 50
   
   enable_script_security
   script_user root
}

vrrp_script check_haproxy {
    script "/etc/keepalived/check_haproxy.sh"
    interval 2
    fall 2
    rise 2
    
    user root
}

vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 66
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp668
    }
    virtual_ipaddress {
        10.10.10.66/24
    }
    
    track_script {                      
        check_haproxy
    }
}
EOF
```

### 日志轮转配置

```properties
sudo tee /etc/logrotate.d/keepalived << 'EOF'
/var/log/keepalived/keepalived.log {
    # 每天轮转一次
    daily
    # 保留最近 180 个轮转文件
    rotate 180
    # 启用压缩（gzip）
    compress    
    # 延迟压缩：上一个轮转文件在下次轮转时才压缩,方便查看最新日志
    delaycompress
    
    # 指定轮转时使用的用户和组
    su syslog adm
    
    # 如果日志文件不存在，不报错
    missingok
    # 如果日志文件为空，不轮转
    notifempty
   
    # 创建新日志文件时设置权限
    create 640 syslog adm
    # 轮转后执行 postrotate 脚本
    postrotate
    # 通知 rsyslog 重新打开日志文件
    systemctl kill -s HUP rsyslog
    endscript
}
EOF
```

### 检查轮转效果

```properties
# 手动触发轮转并查看详情
sudo logrotate -vf /etc/logrotate.d/keepalived
# 查看当前 logrotate 状态
sudo cat /var/lib/logrotate/status | grep keepalived
# 确认文件大小变化
du -sh /var/log/keepalived/keepalived.log*
cat /var/log/keepalived/keepalived.log.1
# 确认 rsyslog 还在正常写日志
sudo tail -f -n 500 /var/log/keepalived/keepalived.log
```

#### 验证是否安装成功

```properties
sudo systemctl daemon-reload
sudo systemctl enable keepalived
sudo systemctl start keepalived
#----重启查看状态--------------------
sudo systemctl restart keepalived
sudo systemctl status keepalived

# 测试日志是否正常写入
sudo systemctl stop haproxy
# 查看结果
tail -f -n 500 /var/log/keepalived.log
```

#### 查看日志文件
```properties
sudo systemctl status keepalived
sudo systemctl is-enabled keepalived 
tail -f -n 500 /var/log/keepalived.log
```












