# openresty-centos离线安装步骤

## openresty 安装
### 1. 需要准备的文件列表

共 12 个文件

```properties
1. openresty-zlib-1.2.11-3.el7.centos.x86_64.rpm
2. openresty-pcre-8.44-1.el7.x86_64.rpm
3. openresty-openssl111-1.1.1k-1.el7.x86_64.rpm
4. openresty-1.19.3.2-1.el7.x86_64.rpm
5. http.lua
6. http_connect.lua
7. http_headers.lua
8. nginx.conf
9. read.lua
10. upload.lua
11. dist_b.zip
12. dist_c.zip
```

### 2. 下载 rpm包

共 4 个, 执行 `yum reinstall --downloadonly --downloaddir=/tmp/rpm XXX`
- openresty-1.19.3.2-1.el7                                                     -
- openresty-openssl111-1.1.1k-1.el7 
- openresty-pcre-8.44-1.el7 
- openresty-zlib-1.2.11-3.el7.centos

### 增加 http 插件

将以下 3 个文件复制到 `/usr/local/openresty/lualib/resty`目录下
- http.lua
- http_connect.lua
- http_headers.lua

### 3. 安装
**顺序有先后**

```bash
rpm -Uvh openresty-zlib-1.2.11-3.el7.centos.x86_64.rpm
rpm -Uvh openresty-pcre-8.44-1.el7.x86_64.rpm
rpm -Uvh openresty-openssl111-1.1.1k-1.el7.x86_64.rpm
rpm -Uvh openresty-1.19.3.2-1.el7.x86_64.rpm
```

### 4. 修改 nginx.conf

直接替换文件 `/usr/local/openresty/nginx/conf/nginx.conf`

### 5. 创建Lua文件

- 在nginx 目录下 创建文件夹 lua
- 添加 2 个文件 `read.lua` 和 `upload.lua`
- 修改 upload.lua 第一行的 另外一台机器的 IP和端口

### 6. 创建 data 文件

- 在nginx 目录下 创建文件夹 data
- 在 data 目录下 创建文件 data.json,  写入内容 `{"list":{}}`,  并修改写权限，执行 `chmod 666 data.json`
- 在 data 目录下 解压 `dist_b.zip` 到目录dist_b，并放入B端前端文件，其中已经加入一层文件夹 zszq30b
- 在 data 目录下 解压 `dist_c.zip` 到目录dist_c，并放入C端前端文件，其中已经加入一层文件夹 zszq30c

### 7. 开启 openresty 服务

```bash
systemctl daemon-reload && systemctl start openresty  && systemctl enable openresty
```

### 8. 日志文件

在 /usr/local/openresty/nginx/logs 文件夹下,

- 错误日志在 error.log
- 访问日志在 access.log

### 9. 访问页面

- 两台机器C端： http://192.168.10.XXX:8001/zszq30c/
- 两台机器B端： http://192.168.10.XXX:9001/zszq30b/
- 重点测试两边B端增删改查，看一下是否对端页面也一并修改

### 10. 重启服务

```bash
systemctl restart openresty
```

---
## 部署 keepalived

```bash
yum install -y gcc openssl-devel popt-devel
mkdir -p /k8s/keepalived && cd /k8s/keepalived
wget http://www.keepalived.org/software/keepalived-2.0.6.tar.gz
tar -zxvf keepalived-2.0.6.tar.gz && rm -f keepalived-2.0.6.tar.gz
cd keepalived-2.0.6 && mkdir keepalived-prefix
# --prefix：指定安装路径
./configure --prefix=$(pwd)/keepalived-prefix
# 安装
make && make install
#复制文件
cp keepalived/etc/init.d/keepalived /etc/init.d
cp keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cd keepalived-prefix/ && mkdir /etc/keepalived
#复制文件
cp etc/keepalived/keepalived.conf /etc/keepalived/
```
默认每隔3秒钟执行一次检测脚本，检查nginx服务是否启动，如果没启动就把nginx服务启动起来，如果启动不成功，就把keepalived服务down掉，让漂浮到备keepalived上
```bash
cat << EOF | tee /etc/keepalived/check_nginx.sh
#!/bin/bash
run=\$(ps -C kube-nginx --no-header | wc -l)
if [ \$run -eq 0 ]; then
  systemctl stop kube-nginx
  systemctl start kube-nginx
  sleep 3
  if [ \$(ps -C kube-nginx --no-header | wc -l) ]; then
    killall keepalived
  fi
fi
EOF
```

检测脚本一定要写在vrrp_instance的前面也就是上面，而且花括号一定要有空格，追踪trace_script一定要在vip的后面，多少人栽在了这上面好多小时

### 1. 在主节点机器上安装master

```bash
chmod a+x /etc/keepalived/check_nginx.sh
cat << EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     pantj@investoday.com.cn
   }
   notification_email_from kaadmin@localhost
   smtp_server 192.168.10.115
   smtp_connect_timeout 30
   router_id vmsrv-010-110
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
        auth_pass investoday0755
    }
    virtual_ipaddress {
        192.168.10.99/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```
### 2. 在备节点机器上安装BACKUP
- router_id：本机hostname
- script "/etc/keepalived/check_nginx.sh" **nginx检测脚本目录**
- state BACKUP **主服务器必须设置为MASTER**
- interface ens192 **与服务器的网卡接口必须一致**
- virtual_router_id 99 **主、备服务器的id必须一致，不能与其他keepalive集群相同，所以采用VIP**
- priority 90 **备份服务器的priority必须小于主服务器的priority**
- auth_pass investoday0755 **服务器的密码必须一致**
- 192.168.10.99/24 **虚拟IP地址，和服务器必须在同一网段并且没有使用**
```bash
chmod a+x /etc/keepalived/check_nginx.sh
cat << EOF | tee /etc/keepalived/keepalived.conf
! Configuration File for keepalived

global_defs {
   notification_email {
     pantj@investoday.com.cn
   }
   notification_email_from kaadmin@localhost
   smtp_server 192.168.10.115
   smtp_connect_timeout 30
   router_id vmsrv-010-111
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
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass investoday0755
    }
    virtual_ipaddress {
        192.168.10.99/24
    }
    
    track_script {                      
        chk_http_port
    }
}
EOF
```
验证是否安装成功，查看日志文件
```bash
# 验证是否安装成功
systemctl daemon-reload && systemctl enable keepalived && systemctl restart keepalived

#查看日志文件
tail -f -n 500 /var/log/messages
```