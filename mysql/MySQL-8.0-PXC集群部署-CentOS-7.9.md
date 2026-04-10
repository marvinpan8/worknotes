# CentOS-7部署MySQL8.0-PXC集群

- PXC官网：https://www.percona.com/mysql/software/percona-xtradb-cluster
- MySQL官网：https://dev.mysql.com/doc/refman/8.0/en/x-plugin-options-system-variables.html
- PXC运维手册：https://www.whyvv.top/2019/07/05/PXC-mysql/


## 版本选择

- CentOS Linux release 7.9.2009 (Core)
- Percona XtraDB Cluster **8.0.37**
- Percona-XtraBackup-8.0.35-31

## 准备三台CentOS 服务器：

| **IP**      | **端口** | **主机名+角色** |
| ----------- | -------- | --------------- |
| 171.17.0.40 | 3306     | pxc1            |
| 171.17.0.41 | 3306     | pxc2            |
| 171.17.0.42 | 3306     | pxc3            |

| **端口**       | **用途**                                                     |
| -------------- | ------------------------------------------------------------ |
| **3306**       | **（数据服务端口）标准 MySQL 端口用于客户端连接和 SST 的mysqldump** |
| **4567**       | **（集群通信端口）处理写入集复制（TCP）和多播复制（UDP）。** |
| 4444（未使用） | （全量同步端口）用于 SST 命令 rsync 和 Percona XtraBackup。  |
| 4568（未使用） | （增量同步端口）促进增量状态转移（IST）以实现节点同步。      |

## 检查端口，创建数据文件夹

```properties
netstat -tunlp |grep 3306  
netstat -tunlp |grep 4567 
netstat -tunlp |grep 4444   
netstat -tunlp |grep 4568
# 创建日志文件夹
mkdir -p /data/mysql/log 
chown -R mysql:mysql /data/mysql
cd /data/mysql/ && ll /data
```



# ■■■ 集群搭建

##  配置host解析(3台设备)
```properties
cat /etc/hosts  
vim /etc/hosts 
-------------------
172.17.0.40 pxc1
172.17.0.41 pxc2 
172.17.0.43 pxc3
```
## 3节点先删除卸载PXC

```properties
# 停止服务
systemctl stop mysql
# 卸载安装程序
yum list | grep percona
yum remove percona-xtradb-cluster
yum remove percona-xtrabackup-80
yum remove percona-toolkit
# 卸载release
yum remove percona-release
# 删除数据和配置文件
rm -rf /var/lib/mysql
rm -f /etc/my.cnf
```

## 3节点官方RPM下载安装（不推荐）

- **缺点：依赖复杂、版本冲突风险高**
- 下载地址：https://www.percona.com/downloads
- 选择Centos7:
  - Percona-XtraDB-Cluster-8.0.37-rd29a325-el7-x86_64-bundle.tar
  - Percona-XtraBackup-8.0.35-31-r55ec21d7-el7-x86_64-bundle.tar
- 解压安装

```properties
tar -xvf  Percona-XtraDB-Cluster-8.0.37-rd29a325-el7-x86_64-bundle.tar
tar -xvf  Percona-XtraBackup-8.0.35-31-r55ec21d7-el7-x86_64-bundle.tar
```

## 3节点官方仓库在线安装（推荐）

官方文档：

https://docs.percona.com/percona-xtradb-cluster/8.0/yum.html#install-from-percona-software-repository

```properties
# 删除 MariaDB 程序包
yum -y remove mari*
yum remove mysql*

yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm 
percona-release enable-only pxc-80 release 
percona-release enable tools release
# 安装PXC核心组件（包含Galera库,速度较慢）
yum install -y percona-xtradb-cluster
```

## 安装后文件的默认位置

| **Files**      | **Location**        | 校验                      |
| -------------- | ------------------- | ------------------------- |
| mysqld server  | /usr/bin            | ll /usr/bin \| grep msyql |
| Configuration  | /etc/my.cnf         | ll /etc/my.cnf            |
| Data directory | /var/lib/mysql      | ll /var/lib/mysql         |
| Logs           | /var/log/mysqld.log | ll /var/log/mysqld.log    |





---

# ■■■【40】节点手动生成100年SSL证书

#### 参考另一篇文章《OpenSSL-1.0证书生成步骤》，证书目录在 /etc/mysql/certs/

```properties
mkdir -p /etc/mysql/certs/ && cd /etc/mysql/certs/
```

#### 复制证书到其它节点

```properties
scp ca.pem pxc2:/etc/mysql/certs/
scp ca.pem pxc3:/etc/mysql/certs/
scp server*.pem pxc2:/etc/mysql/certs/
scp server*.pem pxc3:/etc/mysql/certs/
scp client*.pem pxc2:/etc/mysql/certs/
scp client*.pem pxc3:/etc/mysql/certs/
# 每个节点修改证书文件权限
chown -R mysql:mysql /etc/mysql/certs
```



## 编辑配置文件(3节点)：vim /etc/my.cnf

#### 以下修改所有节点不一致

- 在 MySQL 8.0.21 之前， [`mysqlx_bind_address`](https://dev.mysql.com/doc/refman/8.0/en/x-plugin-options-system-variables.html#sysvar_mysqlx_bind_address) 接受单个地址值
- **从 MySQL 8.0.21 开始**， [`mysqlx_bind_address`](https://dev.mysql.com/doc/refman/8.0/en/x-plugin-options-system-variables.html#sysvar_mysqlx_bind_address) 它既可以接受单个值，也可以接受逗号分隔的值列表

```properties
[mysqld]
server-id=1
bind-address=127.0.0.1,172.17.0.40
mysqlx-bind-address=172.17.0.40
wsrep_node_name=pxc1
wsrep_node_address=172.17.0.40
```
#### 以下修改所有节点一致
```properties
wsrep_cluster_address=gcomm://172.17.0.40:4567,172.17.0.41:4567,172.17.0.42:4567
datadir=/data/mysql/data
log-error=/data/mysql/log/mysqld.log

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

https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-system-index.html

https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-provider-index.html





---

# ■■■【40】单节点启动bootstrap服务：

### 删除包和所有配置（首次跳过）
```properties
systemctl status mysql@bootstrap
systemctl stop mysql@bootstrap
# 删除文件夹
rm -rf /data/mysql
mkdir -p /data/mysql/log
chown -R mysql:mysql /data/mysql
```
### 启动bootstrap

```properties
systemctl start mysql@bootstrap
systemctl status mysql@bootstrap 
netstat -tunlp |grep mysql
netstat -tunlp |grep 3306
netstat -tunlp |grep 4567
# ----以下端口没打开-------
netstat -tunlp |grep 4444   
netstat -tunlp |grep 4568
```

### 查看本机临时密码 
```properties
grep -i password /data/mysql/log/mysqld.log
------------------------------------- 
A temporary password is generated for root@localhost: pZ3DW_o5oHHk
```

### 修改本机临时密码 ：
```properties
mysql --ssl-mode=DISABLED -uroot -p'pZ3DW_o5oHHk'
# 修改密码 
ALTER USER 'root'@'localhost' IDENTIFIED BY 'HappyNewYear2026'; 
```

### 查看集群状态
```properties
show status like 'wsrep_incoming_addresses';
show status like 'wsrep%';
+----------------------------+--------------------------------------+
| Variable_name              | Value                                |
+----------------------------+--------------------------------------+
| wsrep_local_state_uuid     | c2883338-834d-11e2-0800-03c9c68e41ec |
| ...                        | ...                                  |
| wsrep_local_state          | 4                                    |
| wsrep_local_state_comment  | Synced                               |
| ...                        | ...                                  |
| wsrep_cluster_size         | 1                                    |
| wsrep_cluster_status       | Primary                              |
| wsrep_connected            | ON                                   |
| ...                        | ...                                  |
| wsrep_ready                | ON                                   |
+----------------------------+--------------------------------------+
40 rows in set (0.01 sec)
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

systemctl start mysql
tail -f -n 500 /data/mysql/log/mysqld.log
# 查看集群
systemctl status mysql 
netstat -tunlp |grep mysql
netstat -tunlp |grep 3306
netstat -tunlp |grep 4567
# ----以下端口没打开-------
netstat -tunlp |grep 4444   
netstat -tunlp |grep 4568
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
# 查看集群地址
show status like 'wsrep_incoming_addresses';
+--------------------------+----------------------------------------------------+
| Variable_name            | Value                                              |
+--------------------------+----------------------------------------------------+
| wsrep_incoming_addresses | 172.17.0.41:3306,172.17.0.40:3306,172.17.0.42:3306 |
+--------------------------+----------------------------------------------------+
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
cat /data/mysql/data/gvwstate.dat
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
cat /data/mysql/data/grastate.dat
# ------输出----------------------------
# GALERA saved state
version: 2.1
uuid:    6d340ef5-68f3-11f0-9ecf-fb626157e3f7
seqno:   -1
safe_to_bootstrap: 0
```





# ■■■ 【40】节点关闭引导程序，启动mysql程序
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
systemctl start mysql
tail -f -n 500 /data/mysql/log/mysqld.log
# 查看集群
systemctl status mysql 
netstat -tunlp |grep mysql
netstat -tunlp |grep 3306
netstat -tunlp |grep 4567
# ----以下端口没打开-------
netstat -tunlp |grep 4444   
netstat -tunlp |grep 4568
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
```





---

# ■■■ 用户权限设置

### 修改root用户访问权限，分别登录3节点校验数据

```properties
# 登录
mysql -uroot -p'HappyNewYear2026'

use mysql; 
SELECT host, user FROM user;

UPDATE user SET host = '%' WHERE user = 'root';
flush privileges; 
```

### 创建新用户, 可以使用通配符%

```properties
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
# 删除用户
DROP USER 'cx_user'@'10.244.%';
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
mysql> show grants for 'cx_user'@'10.244.%';
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

### 创建角色并赋予权限（新版本8才有，跳过）

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

### 设置密码过期时间

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





---

#  ■■■ 部署 nginx高可用

#### vim /k8s/nginx/conf/kube-nginx.conf

```properties
    stream {
    
        upstream msyql_tcp {
            hash $remote_addr consistent;
            server 172.17.0.40:3306 max_fails=3 fail_timeout=30s;
            server 172.17.0.41:3306 max_fails=3 fail_timeout=30s;
            server 172.17.0.42:3306 max_fails=3 fail_timeout=30s;
        }

        server {
            listen 13306;
            proxy_connect_timeout 2s;
            proxy_timeout 120;
            proxy_pass msyql_tcp;
        }
    
    }
```
#### 重启 kube-nginx 服务：

```properties
systemctl restart kube-nginx
```

#### 检查 kube-nginx 服务运行状态

```properties
systemctl status kube-nginx
netstat -anp |grep nginx
```

确保状态为 `active (running)`，否则到 master 节点查看日志，排查原因：

```properties
journalctl -f -u kube-nginx
```

