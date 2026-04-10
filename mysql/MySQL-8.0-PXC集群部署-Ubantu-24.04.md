# Ubantu-24.04部署MySQL-8.0-PXC集群

- PXC官网：https://www.percona.com/mysql/software/percona-xtradb-cluster
- MySQL官网：https://dev.mysql.com/doc/refman/8.0/en/x-plugin-options-system-variables.html
- PXC运维手册：https://www.whyvv.top/2019/07/05/PXC-mysql/


## 版本选择

- Ubuntu 24.04.4 LTS (noble) ------ lsb_release -a
- Percona XtraDB Cluster **8.4**
- Percona-XtraBackup **8.4.7-7-1**

## 准备三台服务器：

| **IP**       | **端口** | **主机名+角色** |
| ------------ | -------- | --------------- |
| 10.10.20.201 | 3306     | pxc1            |
| 10.10.20.202 | 3306     | pxc2            |
| 10.10.20.203 | 3306     | pxc3            |

| **端口**       | **用途**                                                     |
| -------------- | ------------------------------------------------------------ |
| **3306**       | **（数据服务端口）标准 MySQL 端口用于客户端连接和 SST 的mysqldump** |
| **33060** | （X Plugin 监听端口）X Plugin。 |
| **4567**       | **（集群通信端口）处理写入集复制（TCP）和多播复制（UDP）。** |
| 4444（未使用） | （全量同步端口）用于 SST 命令 rsync 和 Percona XtraBackup。  |
| 4568（未使用） | （增量同步端口）促进增量状态转移（IST）以实现节点同步。      |


## 检查端口，创建数据文件夹

```properties
sudo apt update
sudo apt install net-tools -y

sudo netstat -tunlp |grep 3306  
sudo netstat -tunlp |grep 4567 
sudo netstat -tunlp |grep 4444   
sudo netstat -tunlp |grep 4568
# 创建日志文件夹
sudo mkdir -p /data/mysql/log 
sudo chown -R mysql:mysql /data/mysql
cd /data/mysql/ && ll /data
```
## 更换国内源

```properties
sudo vim /etc/apt/sources.list.d/ubuntu.sources
# 修改成
URIs: http://mirrors.aliyun.com/ubuntu/
sudo apt update && sudo apt upgrade
```
## 添加 apt 代理
```properties
sudo vim /etc/apt/apt.conf.d/95proxies
#--------------------------------------
Acquire::http::Proxy "http://10.10.20.201:7897";
Acquire::https::Proxy "http://10.10.20.201:7897";
```

## 禁用 防火墙 和 AppArmo

```properties
sudo systemctl stop firewalld && sudo systemctl disable firewalld
sudo apparmor_status
sudo systemctl status apparmor
sudo systemctl stop apparmor
sudo systemctl disable apparmor
```

## 永久关闭 swap

```properties
# 临时关闭
sudo swapoff -a 
# 永久关闭
sudo vim /etc/fstab
# 注释了下面这一行
#/swap.img	none	swap	sw	0	0

# 校验
free
```

## 安装时间同步服务器 chrony 

- 201 服务器，202/203客户端

```properties
sudo apt install chrony
sudo vim /etc/chrony/chrony.conf
# 删除 4 行 pool 行内容, 添加下面的ntp服务器
server ntp.aliyun.com iburst
server ntp.tencent.com iburst
server ntp.tuna.tsinghua.edu.cn iburst
server cn.pool.ntp.org iburst
server ntp.ntsc.ac.cn iburst

#-- 其他客户端 配置----------------------
server 10.10.20.201 iburst
# 在最后面添加下面内容
allow 0.0.0.0/0
local stratum 10
#--------------------------------------
sudo systemctl daemon-reload
sudo systemctl restart chronyd
sudo systemctl enable --now chrony
# 检查 123 端口
ss -ntul |grep 123
sudo systemctl status chrony
# 检查偏差，应该接近0
sudo chronyc tracking | grep "System time"
# 立即强制同步
sudo chronyc makestep
```

## 打开 ipvs

```properties
sudo apt install ipvsadm -y
# 创建 ipvs.conf
sudo tee /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
ip_vs_lc
nf_conntrack
EOF
# 重启后会自动加载，也可以立即测试
sudo systemctl restart systemd-modules-load
lsmod | grep ip_vs
sudo ipvsadm -L -n
```



# ■■■ 集群搭建

##  配置host解析(3台设备)
```properties
sudo cat /etc/hosts  
sudo vim /etc/hosts 
-------------------
10.10.20.201 pxc1
10.10.20.202 pxc2 
10.10.20.203 pxc3
```
## 3节点先删除卸载PXC

```properties
# 停止服务
sudo systemctl status mysql
sudo systemctl stop mysql
# 卸载安装程序
apt list --installed | grep percona
sudo apt remove --purge percona-xtradb-cluster
sudo apt remove --purge percona-xtrabackup-80
sudo apt remove --purge percona-toolkit
# 卸载release
sudo apt remove --purge percona-release
# 删除数据和配置文件
rm -rf /var/lib/mysql
rm -f /etc/my.cnf
```
## 3节点官方仓库在线安装（推荐）

官方文档：

https://docs.percona.com/percona-xtradb-cluster/8.4/apt.html#install-from-repository

下载存储库软件包

https://repo.percona.com/apt/percona-release_latest.generic_all.deb

```properties
# 删除 MariaDB 程序包
sudo apt -y remove mariadb*
sudo apt remove mysql*

# 安装必要的软件包
sudo apt install -y wget gnupg2 lsb-release curl
# 下载存储库软件包
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
# 安装软件包dpkg：
sudo dpkg -i percona-release_latest.generic_all.deb
# 刷新本地缓存以更新软件包信息：
sudo apt update
# 启用 PXC 安装源
sudo percona-release setup pxc-84-lts
ll /etc/apt/sources.list.d/percona*
# 安装集群：（包含Galera库,速度较慢）
sudo apt install -y percona-xtradb-cluster
#------默认的安装软件------------------------------------------
percona-xtradb-cluster-common amd64 1:8.4.7-7-1.noble [8,508 B]
percona-xtradb-cluster-client amd64 1:8.4.7-7-1.noble [18.6 MB]                           percona-xtradb-cluster-server amd64 1:8.4.7-7-1.noble [146 MB]                           percona-xtradb-cluster amd64 1:8.4.7-7-1.noble [3,676 B] 
percona-telemetry-agent amd64 1.0.9-1.noble [11.0 MB]                                     qpress amd64 11-4.noble [40.5 kB]  
```

## 安装后文件的默认位置

| **Files**      | **Location**        | 校验                           |
| -------------- | ------------------- | ------------------------------ |
| mysqld server  | /usr/bin            | sudo ls /usr/bin \| grep msyql |
| Configuration  | /etc/my.cnf         | sudo ls /etc/my.cnf            |
| Data directory | /var/lib/mysql      | sudo ls /var/lib/mysql         |
| Logs           | /var/log/mysqld.log | sudo ls /var/log/mysqld.log    |



---

# ■■■【201】节点手动生成100年SSL证书

#### 参考另一篇文章《OpenSSL-3.0证书生成步骤》，证书目录在 /etc/mysql/certs/

```properties
sudo mkdir -p /etc/mysql/certs/ && cd /etc/mysql/certs/
```

#### 复制证书到其它节点

```properties
scp ca.pem pxc2:/etc/mysql/certs/
scp ca.pem pxc3:/etc/mysql/certs/
scp server*.pem pxc2:/etc/mysql/certs/
scp server*.pem pxc3:/etc/mysql/certs/
scp client*.pem pxc2:/etc/mysql/certs/
scp client*.pem pxc3:/etc/mysql/certs/
# 每个节点修改证书文件权限为 mysql 用户
sudo chown -R mysql:mysql /etc/mysql/certs
```



## 编辑配置文件(3节点)  mysqld.cnf

#### sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf

#### 以下修改所有节点不一致

`mysqlx-bind-address` 是 **X Plugin**（负责 MySQL 的 X Protocol 连接）的系统变量，用于指定该插件监听 TCP/IP 连接的网络地址。默认端口 **33060**的连接：

- 在 MySQL 8.0.21 之前， [`mysqlx_bind_address`](https://dev.mysql.com/doc/refman/8.0/en/x-plugin-options-system-variables.html#sysvar_mysqlx_bind_address) 接受单个地址值
- **从 MySQL 8.0.21 开始**， [`mysqlx_bind_address`](https://dev.mysql.com/doc/refman/8.0/en/x-plugin-options-system-variables.html#sysvar_mysqlx_bind_address) 它既可以接受单个值，也可以接受逗号分隔的值列表

```properties
[mysqld]
server-id=1
bind-address=127.0.0.1,10.10.20.201
mysqlx-bind-address=10.10.20.201
wsrep_node_name=pxc1
wsrep_node_address=10.10.20.201

skip-name-resolve
proxy_protocol_networks=10.10.20.201,10.10.20.202,10.10.20.203
```
#### 以下修改所有节点一致
```properties
wsrep_cluster_address=gcomm://10.10.20.201:4567,10.10.20.202:4567,10.10.20.203:4567
datadir=/data/mysql/data
log-error=/data/mysql/log/error.log

[client]
#----------- add my config-----------------
ssl-ca=/etc/mysql/certs/ca.pem
ssl-cert=/etc/mysql/certs/client-cert.pem
ssl-key=/etc/mysql/certs/client-key.pem

[mysqld]
ssl-key=/etc/mysql/certs/server-key.pem
ssl-ca=/etc/mysql/certs/ca.pem
ssl-cert=/etc/mysql/certs/server-cert.pem
wsrep_provider_options=”socket.ssl_key=/etc/mysql/certs/server-key.pem;socket.ssl_cert=/etc/mysql/certs/server-cert.pem;socket.ssl_ca=/etc/mysql/certs/ca.pem”

[sst]
encrypt=4
ssl-key=/etc/mysql/certs/server-key.pem
ssl-ca=/etc/mysql/certs/ca.pem
ssl-cert=/etc/mysql/certs/server-cert.pem
```

### 查看配置

```properties
mysqld --verbose --help
```

官方配置说明：

https://docs.percona.com/percona-xtradb-cluster/8.4/wsrep-system-index.html

https://docs.percona.com/percona-xtradb-cluster/8.4/wsrep-provider-index.html



---

# ■■■【201】单节点启动bootstrap服务：

### 所有关机时 先启动的引导服务节点是如下状态

- safe_to_bootstrap: 1
- seqno 最大

```properties
sudo cat /data/mysql/data/grastate.dat
# GALERA saved state
version: 2.1
uuid:    639978e1-237a-11f1-bb5b-bb6bfebcee24
seqno:   279
safe_to_bootstrap: 1
```

### ~~删除包和所有配置（首次跳过）~~
```properties
sudo systemctl status mysql
sudo systemctl stop mysql
sudo systemctl status mysql@bootstrap
sudo systemctl stop mysql@bootstrap
# 删除文件夹
sudo rm -rf /data/mysql
sudo mkdir -p /data/mysql/log
sudo chown -R mysql:mysql /data/mysql
```
### 启动bootstrap

```properties
sudo systemctl start mysql@bootstrap
sudo systemctl status mysql@bootstrap 
sudo netstat -tunlp |grep mysql
# ----打开 3306 33060 4567 没打开 4444 4568-------
```

### 查看本机临时密码 
```properties
sudo grep -i password /data/mysql/log/error.log
------------------------------------- 
A temporary password is generated for root@localhost: (J)#hVSij6gG
```

### 修改本机临时密码 ：
```properties
mysql --ssl-mode=DISABLED -uroot -p'(J)#hVSij6gG'
# 修改密码 
ALTER USER 'root'@'localhost' IDENTIFIED BY 'HappyNewYear2026'; 
```

### 查看集群状态
```properties
# 查看执行引擎 InnoDB
SHOW VARIABLES LIKE 'default_storage_engine';
show status like 'wsrep_incoming_addresses';
show status where Variable_name in ('wsrep_cluster_size','wsrep_cluster_status','wsrep_connected','wsrep_ready') ;
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_size   | 1       |
| wsrep_cluster_status | Primary |
| wsrep_connected      | ON      |
| wsrep_ready          | ON      |
+----------------------+---------+
```
- **wsrep_cluster_size： 集群几个节点**
- **wsrep_cluster_status：Primary 主组件**
- **wsrep_connected：ON  已连接**
- **wsrep_ready：ON  准备好进行写集复制。**



---

# ■■■ 依次启动第2、3个节点加入集群

> **注意：**
>
> - 如果节点状态为`Joiner`，则表示 SST 尚未完成。
> - 在所有其他节点都处于`Synced`状态之前，请勿添加新节点。

```properties
# 先删除文件夹
rm -rf /data/mysql
mkdir -p /data/mysql/log
chown -R mysql:mysql /data/mysql

sudo systemctl start mysql
sudo tail -f -n 500 /data/mysql/log/error.log
# 查看集群
sudo systemctl status mysql 
sudo netstat -tunlp |grep mysql
# ----打开 3306 33060 4567 没打开 4444 4568-------
```
### 校验PXC集群结果

```properties
# 登录
mysql -uroot -p'HappyNewYear2026'
# 查看
select * from performance_schema.pxc_cluster_view;
+-----------+--------------------------------------+--------+-------------+---------+
| HOST_NAME | UUID                                 | STATUS | LOCAL_INDEX | SEGMENT |
+-----------+--------------------------------------+--------+-------------+---------+
| pxc2      | 4548d8de-68f4-11f0-ad40-ee39119b9036 | SYNCED |           0 |       0 |
| pxc1      | 6d3387a7-68f3-11f0-a37b-1b1b4f481a74 | SYNCED |           1 |       0 |
| pxc3      | e71489bc-68f6-11f0-b9ee-b328c85635d9 | SYNCED |           2 |       0 |
+-----------+--------------------------------------+--------+-------------+---------+
# 查看集群地址, 为空也正常？
show status like 'wsrep_incoming_addresses';
```
### 校验wsrep集群结果

```properties
# 查看集群变量
show status where Variable_name in ('wsrep_cluster_size','wsrep_cluster_status','wsrep_connected','wsrep_ready') ;
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_size   | 3       |
| wsrep_cluster_status | Primary |
| wsrep_connected      | ON      |
| wsrep_ready          | ON      |
+----------------------+---------+
4 rows in set (0.01 sec)
```
- wsrep_cluster_size： 集群几个节点
- wsrep_cluster_status：Primary 主组件
- wsrep_connected：ON  已连接
- wsrep_ready：ON  准备好进行写集复制。

### 重要文件

#### gvwstate.dat：节点关闭前的集群状态

```properties
sudo cat /data/mysql/data/gvwstate.dat
# ------输出----------------------------
my_uuid: cffe94d7-6903-11f0-9972-4b0adc7b3482
#vwbeg
view_id: 3 4548d8de-68f4-11f0-ad40-ee39119b9036 5
bootstrap: 0
member: 4548d8de-68f4-11f0-ad40-ee39119b9036 0
member: cffe94d7-6903-11f0-9972-4b0adc7b3482 0
member: e71489bc-68f6-11f0-b9ee-b328c85635d9 0
#vwend
```
#### grastate.dat

```properties
sudo cat /data/mysql/data/grastate.dat
# ------输出----------------------------
# GALERA saved state
version: 2.1
uuid:    6d340ef5-68f3-11f0-9ecf-fb626157e3f7
seqno:   -1
safe_to_bootstrap: 0
```



# ■■■ 【201】节点关闭引导程序，启动mysql程序
PXC 集群允许动态下线节点，但需要注意的是节点的启动命令和关闭命令必须一致，

- **所有节点mysql@bootstrap 引导程序都没有自启动**
- **所有节点mysql 程序都是自启动**

```properties
systemctl is-enabled mysql@bootstrap
# disabled
systemctl is-enabled mysql
# enabled
```

### 启动MySQL程序

```properties
sudo systemctl status mysql@bootstrap
sudo systemctl stop mysql@bootstrap
sudo systemctl start mysql
sudo tail -f -n 500 /data/mysql/log/error.log
# 查看集群
sudo systemctl status mysql 
sudo netstat -tunlp |grep mysql
# ----打开 3306 33060 4567 没打开 4444 4568-------
```

### 登录其他节点校验PXC集群结果

```properties
# 登录
mysql -uroot -p'HappyNewYear2026'
# 查看
select * from performance_schema.pxc_cluster_view;
+-----------+--------------------------------------+--------+-------------+---------+
| HOST_NAME | UUID                                 | STATUS | LOCAL_INDEX | SEGMENT |
+-----------+--------------------------------------+--------+-------------+---------+
| pxc2      | 4548d8de-68f4-11f0-ad40-ee39119b9036 | SYNCED |           0 |       0 |
| pxc1      | 6d3387a7-68f3-11f0-a37b-1b1b4f481a74 | SYNCED |           1 |       0 |
| pxc3      | e71489bc-68f6-11f0-b9ee-b328c85635d9 | SYNCED |           2 |       0 |
+-----------+--------------------------------------+--------+-------------+---------+
```
### 登录其他节点校验wsrep集群结果

```properties
# 查看集群变量
show status where Variable_name in ('wsrep_cluster_size','wsrep_cluster_status','wsrep_connected','wsrep_ready') ;
+----------------------+---------+
| Variable_name        | Value   |
+----------------------+---------+
| wsrep_cluster_size   | 3       |
| wsrep_cluster_status | Primary |
| wsrep_connected      | ON      |
| wsrep_ready          | ON      |
+----------------------+---------+
4 rows in set (0.01 sec)

# 查看默认字符集和排序规则
SHOW VARIABLES LIKE 'character_set_server';
SHOW VARIABLES LIKE 'collation_server';
```





---

# ■■■ 用户权限设置

### 修改root用户访问权限，分别登录3节点校验数据

```properties
# 登录
mysql -uroot -p'HappyNewYear2026'

use mysql; 
SELECT host, user, plugin FROM mysql.user;
# 创建并授权 VPN 访问用户
CREATE USER 'root'@'10.100.0.%' IDENTIFIED BY 'HappyNewYear2026';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'10.100.0.%' WITH GRANT OPTION;
show grants for 'root'@'10.100.0.%';
# 创建并授权节点间访问用户
CREATE USER 'root'@'pxc%' IDENTIFIED BY 'HappyNewYear2026';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'pxc%' WITH GRANT OPTION;
SHOW GRANTS FOR 'root'@'pxc%';
# UPDATE user SET host = '10.100.0.%' WHERE user = 'root' and host = '10.10.20.%';
flush privileges; 
# DELETE FROM user WHERE user='root' and host='pxc%';
## 查看当前会话已启用的角色
SELECT * FROM information_schema.ENABLED_ROLES;
## 查看当前用户可用的角色
SELECT * FROM information_schema.APPLICABLE_ROLES;
```

### 创建新用户, 可以使用通配符%或_

```properties
CREATE USER 'keymgr_user'@'pxc%' IDENTIFIED BY 'VsjX5nEcvymRaJ36';
GRANT ALL PRIVILEGES ON `keymgr-prod`.* TO 'keymgr_user'@'pxc%';
FLUSH PRIVILEGES;


# 允许用户客户端的IP地址 10.244.开头能够访问
CREATE USER 'cx_user'@'10.244.%' IDENTIFIED BY 'cx_admin';
CREATE USER 'cx_user'@'localhost' IDENTIFIED BY 'cx_admin';
# 创建主机名用户
CREATE USER 'cx_user'@'node01' IDENTIFIED BY 'cx_admin';
CREATE USER 'cx_user'@'node02' IDENTIFIED BY 'cx_admin';
CREATE USER 'cx_user'@'node03' IDENTIFIED BY 'cx_admin';

# 查看
use mysql; 
SELECT host, user FROM user;
show databases;
show tables;

# 查看自己权限
show grants
# 查看特定用户权限
show grants for 'cx_user'@'10.244.%';

# 修改密码 
ALTER USER 'cx_user'@'10.244.%' IDENTIFIED BY 'newpassword'; 
```

### 删除用户
```properties
DROP USER 'keymgr_user'@'10.10.10.99';
```



### 授予权限

- **在 MySQL 8.0 中，不能在 GRANT 命令中同时修改用户密码和授予权限。需要分两步完成：**
  **！！！！但使用旧版本的命令会报错！！！！！！**

```properties
# 创建数据库
CREATE DATABASE IF NOT EXISTS `powerjob-daily` DEFAULT CHARSET utf8mb4;
# 创建表  
other/powerjob-mysql.sql

# 授予某数据库的所有权限
use powerjob-daily;
GRANT all privileges ON `powerjob-daily`.* TO 'cx_user'@'10.244.%' WITH GRANT OPTION;
GRANT all privileges ON `powerjob-daily`.* TO 'cx_user'@'localhost' WITH GRANT OPTION;

# 设置主机名权限
GRANT all privileges ON `powerjob-daily`.* TO 'cx_user'@'node01' WITH GRANT OPTION;
GRANT all privileges ON `powerjob-daily`.* TO 'cx_user'@'node02' WITH GRANT OPTION;
GRANT all privileges ON `powerjob-daily`.* TO 'cx_user'@'node03' WITH GRANT OPTION;

flush privileges;
SELECT host, user FROM user;

# 查看权限
show grants for 'cx_user'@'10.244.%';
+--------------------------------------------------------------------------------------+
| Grants for cx_user@10.244.%                                                          |
+--------------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `cx_user`@`10.244.%`                                           |
| GRANT ALL PRIVILEGES ON `powerjob-daily`.* TO `cx_user`@`10.244.%` WITH GRANT OPTION |
+--------------------------------------------------------------------------------------+
2 rows in set (0.00 sec)
# USAGE ON *.* 表示没有权限，与之相对的是 ALL PRIVILEGES ON *.* 与所有权限。
# GRANT USAGE ON *.* TO 'obge'@'%' 就可以理解为，在任意数据库和任意表上对任何东西没有权限。
# 登录测试
mysql -ucx_user -pcx_admin
use powerjob-daily;
show tables;
```

### 创建只读角色并赋予权限（新版本8才有）

```properties
use mysql;
SELECT ROLE_NAME FROM information_schema.user_roles WHERE GRANTEE LIKE 'cx_user'@'10.244.%';

# 创建只读角色
CREATE ROLE readonly_role;
GRANT SELECT ON `powerjob-daily`.* TO readonly_role;

# 将角色分配给用户
GRANT readonly_role TO `cx_user`@`10.244.%`;
SHOW GRANTS FOR `cx_user`@`10.244.%`;
# 删除角色
DROP ROLE readonly_role;

# 刷新权限：
FLUSH PRIVILEGES;


# MySQL8预设角色，如 READ_ONLY_ADMIN 和 READ_ONLY_REPLICA
GRANT 'READ_ONLY_ADMIN' TO 'username'@'host';
show variables like 'activate_all_roles_on_login';
+-----------------------------+-------+
| Variable_name               | Value |
+-----------------------------+-------+
| activate_all_roles_on_login | OFF   |
+-----------------------------+-------+
```

### 回收撤销权限：revoke【重点】

```properties
revoke 权限1,权限2,…权限n ON 数据库名.表名 from '用户'@'IP地址';
```

- 被回收的权限必须存在，否则会出错

- 整个数据库，使用 ON datebase.*；

- 特定的表：使用 ON datebase.table；

- ALL PRIVILEGES：是一个特殊的权限集合，它指的是用户可以拥有的所有可能的权限。

```properties
# 1.移除用户【全局权限】：
revoke select,delete on *.* from 'tom'@'192.168.137.10';
revoke all privileges on *.* from 'tom'@'192.168.137.10';
 
# 2.移除用户对【特定数据库】的权限：
revoke alter,update on test.* from 'tom'@'192.168.137.10';
revoke all privileges on test.* from 'tom'@'192.168.137.10';
 
# 3.移除用户对【特定表】的权限：
revoke alter,update on test.t1 from 'tom'@'192.168.137.10';
revoke all privileges on test.t1 from 'tom'@'192.168.137.10';
 
# 4.在执行了 REVOKE 语句之后，需要执行以下命令来使权限变更立即生效：
flush privileges;
```

### 设置密码过期时间，默认0永久不过期

```properties
# 设置全局，设置密码每隔90天过期
mysql> set global default_password_lifetime = 90;
 
mysql> show VARIABLES like 'default_password_lifetime';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| default_password_lifetime | 90    |
+---------------------------+-------+
```

### 权限控制机制

```properties
# 四张表：user  db  tables_priv columns_priv
1.用户认证
  查看mysql.user表
2.权限认证
  以select权限为例：
  1.先看 user表里的select_priv权限
   Y:不会接着查看其他的表  拥有查看所有库所有表的权限
   N:接着看db表
  2.db表:  #某个用户对一个数据库的权限。
   Y:不会接着查看其他的表  拥有查看所有库所有表的权限
   N:接着看tables_priv表
3.tables_priv表：     # 针对表的权限
   tables_priv:如果这个字段的值里包括select  拥有查看这张表所有字段的权限，不会再接着往下看了
   tables_priv:如果这个字段的值里不包括select，接着查看下张表还需要有column_priv字段权限
4.columns_priv:       #针对数据列的权限表
   columns_priv:有select，则只对某一列有select权限
    没有则对所有库所有表没有任何权限
   注：其他权限设置一样。
 
# 授权级别排列
- mysql.user #全局授权
- mysql.db   #数据库级别授权
- 其他       #表级，列级授权
```



# ■■■ 部署 proxysql 高可用

Wiki:  https://github.com/sysown/proxysql/wiki#installation

官网安装：https://www.proxysql.com/documentation/installing-proxysql

GitHub: https://github.com/sysown/proxysql/releases

### 升级 openssl 最新版

```properties
# 1. 安装编译依赖
sudo apt update
sudo apt install -y build-essential checkinstall zlib1g-dev

# 2. 下载源码并编译
cd /usr/local/src
sudo wget https://www.openssl.org/source/openssl-3.5.6.tar.gz
tar -zxvf openssl-3.5.6.tar.gz
rm -f openssl-3.5.6.tar.gz
cd openssl-3.5.6

# 3. 配置（安装到独立目录，不覆盖系统版本）
sudo mkdir /usr/local/openssl
ls /usr/local/openssl
sudo ./config --prefix=/usr/local/openssl --openssldir=/usr/local/openssl --libdir=lib shared zlib

# 4. 编译安装
sudo make -j$(nproc)
sudo make install

# 5. 更新动态库缓存
sudo cat /etc/ld.so.conf.d/openssl.conf
echo "/usr/local/openssl/lib" | sudo tee /etc/ld.so.conf.d/openssl.conf
sudo ldconfig

# 6. 验证动态链接器是否找到了正确的库文件
ls /usr/local/openssl/lib
ldd /usr/local/openssl/bin/openssl | grep ssl

# 7. 配置环境变量
sudo cat /etc/profile.d/openssl.sh
echo 'export PATH=/usr/local/openssl/bin:$PATH' | sudo tee /etc/profile.d/openssl.sh
source /etc/profile.d/openssl.sh

# 8. 验证版本
openssl version
```

### 安装 proxysql


```properties
wget https://github.com/sysown/proxysql/releases/download/v3.0.7/proxysql_3.0.7-ubuntu24_amd64.deb
sudo apt install ./proxysql_3.0.7-ubuntu24_amd64.deb
# 版本
proxysql -V
#---------------------------------------------------
ProxySQL version 3.0.7-369-gd7a26b7, codename Truls
# 创建数据文件夹
ls -l /data/proxysql
sudo  mkdir -p /data/proxysql/logs
sudo chown -R proxysql:proxysql /data/proxysql
```
#### sudo vim /etc/proxysql.cnf

- pgsql_ifaces 关闭不了，暂时用 127.0.0.1:6132？

```properties
datadir="/data/proxysql"
errorlog="/data/proxysql/logs/proxysql.log"

admin_variables=
{
    admin_credentials="admin:ekemp2026;cluster_user:ekemp2026"
    mysql_ifaces="127.0.0.1:6032;10.10.20.201:6032"
    pgsql_ifaces="127.0.0.1:6132"
    cluster_username="cluster_user"
    cluster_password="ekemp2026"
}

mysql_variables=
{
    threads=4
    max_connections=2048
    default_query_delay=0
    default_query_timeout=36000000
    have_compress=true
    interfaces="10.10.20.201:6033"
    server_version="8.4.7"
    monitor_username="monitor"
    monitor_password="Fv9S494dZLbcOfVc"
}
pgsql_variables=
{
    interfaces=""
}
```

### 修改 systemd 服务文件

```properties
# 1. 编辑 systemd 服务文件
sudo vim /usr/lib/systemd/system/proxysql.service
# 修改为你的 datadir 路径
PIDFile=/data/proxysql/proxysql.pid
```

### 启动 proxysql  服务

```properties
sudo systemctl daemon-reload
sudo systemctl start proxysql
sudo systemctl stop proxysql
# 检查 proxysql 服务运行状态
sudo systemctl status proxysql
ps -ef |grep proxysql
sudo netstat -anp |grep proxysql
journalctl -f -u proxysql
```

### 修改监听地址

```properties
# 管理员登录
mysql -u admin -pekemp2026 -h 127.0.0.1 -P6032 --prompt='Admin> ' --default-auth=mysql_native_password --ssl-mode=DISABLED

SHOW VARIABLES LIKE 'pgsql-interfaces%';
# 确认 mysql 监听地址
SELECT 'RUNTIME' as layer, variable_value FROM runtime_global_variables WHERE variable_name='mysql-interfaces'
UNION ALL
SELECT 'MEMORY', variable_value FROM global_variables WHERE variable_name='mysql-interfaces'
UNION ALL
SELECT 'DISK', variable_value FROM disk.global_variables WHERE variable_name='mysql-interfaces';
# 确认 mysql 监听地址
SELECT 'RUNTIME' as layer, variable_value FROM runtime_global_variables WHERE variable_name='pgsql-interfaces'
UNION ALL
SELECT 'MEMORY', variable_value FROM global_variables WHERE variable_name='pgsql-interfaces'
UNION ALL
SELECT 'DISK', variable_value FROM disk.global_variables WHERE variable_name='pgsql-interfaces';
```

###  3节点安装原生集群

```properties
SELECT * FROM proxysql_servers;
# 先清空再添加（确保没有脏数据）
DELETE FROM proxysql_servers;
# 配置 proxysql_servers 原生集群
INSERT INTO proxysql_servers (hostname, port, weight, comment) VALUES 
('10.10.20.201', 6032, 1, 'node1'),
('10.10.20.202', 6032, 1, 'node2'),
('10.10.20.203', 6032, 1, 'node3');
# 即时生效
LOAD PROXYSQL SERVERS TO RUNTIME;
SAVE PROXYSQL SERVERS TO DISK;

# 设置集群通信的用户名密码
SELECT * FROM global_variables WHERE variable_name='admin-cluster_username';
# 关键：这个用户必须存在于 admin-admin_credentials 中
SELECT * FROM global_variables WHERE variable_name='admin-admin_credentials';

# 查看各节点的配置版本和校验和
SELECT hostname, port, name, version, epoch, checksum FROM stats_proxysql_servers_checksums ORDER BY hostname, name;
```

### 创建 PXC 监控用户

https://proxysql.com/documentation/the-admin-schemas/monitor-schema

```properties
# PXC创建 监控用户 USAGE 权限
CREATE USER 'monitor'@'10.10.20.20_' IDENTIFIED BY 'Fv9S494dZLbcOfVc';
GRANT USAGE, REPLICATION CLIENT ON *.* TO 'monitor'@'10.10.20.20_';
FLUSH PRIVILEGES;

# 查看监控用户
SHOW TABLES FROM monitor;
SELECT * FROM global_variables WHERE variable_name like 'mysql-monitor%';
# ProxySQL 创建 监控用户
UPDATE global_variables SET variable_value='2000'
  WHERE variable_name IN (
    'mysql-monitor_connect_interval',
    'mysql-monitor_ping_interval',
    'mysql-monitor_read_only_interval'
  );

LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;

# 设置监控模块使用 PROXY 协议连接后端
SELECT * FROM global_variables WHERE variable_name like 'mysql-monitor_proxy_protocol';
SET mysql-monitor_connect_interval = 60000;
# 关键：设置监控模块也发送 PROXY 协议头部
SET mysql-monitor_proxy_protocol = true;
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;


# 查询监控连接
SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 3;
SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 3;

SELECT * FROM monitor.mysql_server_read_only_log ORDER BY time_start_us DESC LIMIT 5;
SELECT * FROM monitor.mysql_server_galera_log ORDER BY time_start_us DESC LIMIT 3;
SELECT * FROM monitor.mysql_server_group_replication_log ORDER BY time_start_us DESC LIMIT 4;
# 验证账户
mysql -u monitor -pFv9S494dZLbcOfVc -h 10.10.20.201 -P 3306 -e "SELECT 1"
```

> 数据库中的所有表`monitor`都是仅追加日志表。为防止无限增长，ProxySQL 会根据配置的历史记录保留设置自动清除旧条目：**MySQL**：此`mysql-monitor_history`变量控制监控记录的保留时间（以微秒为单位）。默认值为`600000`微秒（600 毫秒），这意味着 ProxySQL 会为每个服务器保留该时间窗口内的最新条目。较旧的行将自动删除。
>
> ```properties
> #  设置延长保留  60 seconds of MySQL monitor history
> SET mysql-monitor_history=60000000;
> LOAD MYSQL VARIABLES TO RUNTIME;
> SAVE MYSQL VARIABLES TO DISK;
> ```

### 单节点配置后端服务，看同步

```properties
SELECT * FROM mysql_servers;
# 先清空再添加（确保没有脏数据）
DELETE FROM mysql_servers;
# 添加后端服务器（使用实际 IP）
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES 
(10, '10.10.20.201', 3306),
(10, '10.10.20.202', 3306),
(10, '10.10.20.203', 3306);
# 即时生效
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

# 3节点都是读写节点，不要 配置 Galera 自动分组 每个 ProxySQL 节点手动执行
# 添加应用用户
SELECT * FROM mysql_users;
INSERT INTO mysql_users (username, password, default_hostgroup, active) 
VALUES ('root', 'HappyNewYear2026', 10, 1);
INSERT INTO mysql_users (username, password, default_hostgroup, active) 
VALUES ('keymgr_user', 'VsjX5nEcvymRaJ36', 10, 1);
INSERT INTO mysql_users (username, password, default_hostgroup, active) 
VALUES ('seaweedfs', 'yNh2fD5HnRmUT4xg', 10, 1);
# 即时生效
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

###  透传客户端IP，设置proxy_protocol_networks

```properties
SELECT * FROM global_variables WHERE variable_name like 'mysql-proxy_protocol_networks';
SET mysql-proxy_protocol_networks='10.10.20.0/29';
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;

# 启用扩展进程列表
SET mysql-show_processlist_extended = 1;
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
# 查询从应用通过 ProxySQL 连接信息
SELECT ThreadID, user, cli_host, JSON_EXTRACT(extended_info, '$.src_ip') AS real_client_ip FROM stats_mysql_processlist WHERE extended_info IS NOT NULL;
```

### 验证PXC链接

```properties
# 确认各节点的 hostgroup 分配情况，都是10
SELECT * FROM runtime_mysql_servers;
# 查看 PXC 连接统计
SELECT * FROM stats.stats_mysql_connection_pool;
# 查看 PXC 查询性能
SELECT * FROM stats.stats_mysql_query_digest;
# 查看 PXC 查询规则
SELECT * FROM mysql_query_rules;
```


### 清除 SHUNNED 状态

```properties
SELECT * FROM global_variables WHERE variable_name like 'mysql-shun_recovery_time_sec';
# 设置 SHUNNED 恢复时间为 5 秒（默认约 10 秒）
SET mysql-shun_recovery_time_sec = 5;
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;

# 将所有 SHUNNED 节点恢复为 ONLINE
SELECT * FROM mysql_servers;
SELECT * FROM runtime_mysql_servers;
UPDATE mysql_servers SET status='ONLINE' WHERE status='SHUNNED';
UPDATE runtime_mysql_servers SET status='ONLINE' WHERE status='SHUNNED';

# 加载到运行时
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```


### 删除卸载

```properties
sudo systemctl stop proxysql
sudo systemctl restart proxysql
sudo systemctl status proxysql
# sudo apt purge proxysql
sudo rm -rf /data/proxysql
# 创建数据文件夹
sudo mkdir -p /data/proxysql/logs
sudo chown -R proxysql:proxysql /data/proxysql
# 强制重新初始化
sudo systemctl start proxysql-initial
sudo netstat -anp |grep proxysql
```

---


#  ■■■ 3节点部署 keepalived

### 下载安装 https://www.keepalived.org/download.html

```properties
sudo apt install -y gcc libssl-dev libpopt-dev
mkdir -p /k8s/keepalived && cd /k8s/keepalived
wget http://www.keepalived.org/software/keepalived-2.3.4.tar.gz
tar -zxvf keepalived-2.3.4.tar.gz && rm -f keepalived-2.3.4.tar.gz
cd keepalived-2.3.4 && mkdir keepalived-prefix
# --prefix：指定安装路径
./configure --prefix=$(pwd)/keepalived-prefix
# 编译安装
make && make install
# 复制环境变量配置
sudo cp keepalived/etc/sysconfig/keepalived /etc/default/
# 复制主配置文件
sudo mkdir /etc/keepalived
sudo cp keepalived/etc/keepalived/keepalived.conf.sample /etc/keepalived/
# 复制可执行文件到 PATH
sudo cp bin/keepalived /usr/local/bin/
# 修改 systemd 服务文件
sudo cp keepalived/keepalived.service ./
sudo vim keepalived.service
# 复制 systemd 服务文件
sudo cp keepalived.service /usr/lib/systemd/system/
```
默认每隔3秒钟执行一次检测脚本，检查nginx服务是否启动，如果没启动就把nginx服务启动起来，如果启动不成功，就把keepalived服务down掉，让漂浮到备keepalived上

```properties
sudo tee /etc/keepalived/check_proxysql.sh << EOF
#!/bin/bash
run=\$(ps -C proxysql --no-header | wc -l)
if [ \$run -eq 0 ]; then
  sudo systemctl stop proxysql
  sudo systemctl start proxysql
  sleep 3
  if [ \$(ps -C proxysql --no-header | wc -l) -eq 0 ]; then
    sudo systemctl stop keepalived
  fi
fi
EOF

# 修正执行权限
sudo chmod +x /etc/keepalived/check_proxysql.sh
```

> 注意：检测脚本一定要写在vrrp_instance的前面也就是上面，而且花括号一定要有空格，追踪trace_script一定要在vip的后面，多少人栽在了这上面好多小时



### 【201】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID**：virtual_router_id 改成 98

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
   lvs_id K0S_API
}

vrrp_script check_proxysql {
    script "/etc/keepalived/check_proxysql.sh"
    interval 2
    weight 2
    fall 2
    rise 1
}

vrrp_instance VI_1 {
    state MASTER
    interface bond0
    virtual_router_id 98
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp2026
    }
    virtual_ipaddress {
        10.10.20.99/24
    }
    
    track_script {                      
        chk_nginx
    }
}
EOF
```

#### 【202】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID：virtual_router_id 改成 98**
- **修改priority：80**

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
   lvs_id K0S_API
}

vrrp_script check_proxysql {
    script "/etc/keepalived/check_proxysql.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 98
    priority 80
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp2026
    }
    virtual_ipaddress {
        10.10.20.99/24
    }
    
    track_script {                      
        check_nginx
    }
}
EOF
```
#### 【203】节点

- **查看网卡名称是否为ens192:  ip a** 
- **修改唯一ID：virtual_router_id 改成 98
- **修改priority：60**

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
   lvs_id K0S_API
}

vrrp_script check_proxysql {
    script "/etc/keepalived/check_proxysql.sh"
    interval 2
    weight 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface bond0
    virtual_router_id 98
    priority 60
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ekemp2026
    }
    virtual_ipaddress {
        10.10.20.99/24
    }
    
    track_script {                      
        check_nginx
    }
}
EOF
```


#### 验证是否安装成功
```properties
sudo systemctl daemon-reload
sudo systemctl enable keepalived
sudo systemctl is-enabled keepalived 
sudo systemctl start keepalived
sudo systemctl stop keepalived
```

#### 查看日志文件
```properties
sudo systemctl status keepalived
journalctl -f -u keepalived
```

