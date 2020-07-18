# PG12-Centos7安装

参考官网https://www.postgresql.org/download/linux/redhat/， PostgreSQL Yum Repository 按步骤安装即可

针对PG版本12， centos7_x86_64的安装步骤如下：

## 在线安装

```bash
# Install the repository RPM:
$ yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-42.0-6.noarch.rpm
# Install the client packages:
$ yum install postgresql12
# Optionally install the server packages:
$ yum install postgresql12-server
# Optionally initialize the database and enable automatic start:
$ /usr/pgsql-12/bin/postgresql-12-setup initdb
$ systemctl enable postgresql-12
$ systemctl start postgresql-12
```

## 离线安装

[官网下载地址](<https://yum.postgresql.org/testing/12/redhat/rhel-7-x86_64/>)

```bash
$ yum install pgdg-redhat-repo-42.0-6.noarch.rpm
$ yum install postgresql12-libs-12.2-1PGDG.rhel7.x86_64.rpm
$ yum install postgresql12-12.2-1PGDG.rhel7.x86_64.rpm
$ yum install postgresql12-server-12.2-1PGDG.rhel7.x86_64.rpm
# Optionally initialize the database and enable automatic start:
$ /usr/pgsql-12/bin/postgresql-12-setup initdb
$ systemctl enable postgresql-12
$ systemctl start postgresql-12
```

## 删除

```bash
$ yum remove postgresql12
$ yum remove postgresql12-server
$ yum remove postgresql12-libs
```



## 初始化

默认root并不能连接，需要切换为用户postgres，postgresql在安装时默认添加用户postgres

```
su - postgres
```

进入数据库

```
初始直接：psql
psql -h 192.168.10.225  -p 5432  -U postgres 
```

设置密码：（这里我们设置为postgres）

```
ALTER USER postgres WITH PASSWORD 'postgres';
```

用管理员(postgres)用户建库和创建普通用户

```bash
create user kong with password 'kong';
create database kong owner kong;
grant all privileges on database kong to kong;
```

删除用户

```bash
drop user kong;
```

设置超级用户

```bash
alter role kong superuser;
```

超级用户解决了此问题

```bash
kong=> COMMENT ON EXTENSION plpgsql IS 'PL/pgSQL procedural language';
ERROR:  must be owner of extension plpgsql
```

postgres用某个用户登录，默认登录同名数据库，如果用户名和数据库不同，需要特别指定，否则会报找不到库！**

切换数据库

```bash
#通过\c，后跟数据库和用户名。
$ \c kong kong
#\c后什么都不写会显示当前连接信息：
```

**列出所有库`\l`    列出所有用户`\du`    列出库下所有表`\d`  退出`\q`**

## 修改连接权限

修改配置文件以设置远程登陆

```
vim /var/lib/pgsql/12/data/pg_hba.conf
```

修改前，默认只有本地用户可以访问，所以除了修改ip还要修改权限。如下图示：

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    all             all             ::1/128                 ident
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            ident
host    replication     all             ::1/128                 ident
```

修改后：

- 将METHOD都改为`md5`，此为密码认证方式
- 然后将**后边三行屏蔽掉，新加两行**, 本地IP(192.168.10.225/24，172.30.0.0/16)

```bash
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
#local   replication     all                                     md5
#host    replication     all             127.0.0.1/32            md5
#host    replication     all             ::1/128                 md5
host    all             all            192.168.10.0/24           md5
host    all             all            172.30.0.0/16             md5
```

192.168.10.0/24：表示允许网段192.168.10.0上的所有主机使用所有合法的数据库用户名访问数据库，并提供加密的密码验证。其中，24是子网掩码，表示允许192.168.10.0--192.168.10.255的主机访问。

## 修改远程访问

```bash
vim /var/lib/pgsql/12/data/postgresql.conf
```

将文件中的
#listen_addresses = ‘localhhost’
中的localhost修改为IP(192.168.10.225)

```bash
listen_addresses = '192.168.10.225'     # what IP address(es) to listen on;
                                        # comma-separated list of addresses;
                                        # defaults to 'localhost'; use '*' for all
                                        # (change requires restart)
```

重启

```bash
systemctl restart postgresql-12
```

---

## 数据导入导出

### 命令操作

**数据导出**：

```bash
# pg_dump -U postgres(用户名)  (-t 表名)  数据库名(缺省时同用户名)  > 路径/文件名.sql
$ pg_dump -U postgres -t system_calls wangye > ./test.sql
--------------------------------------------
$ pg_dump -U kong kong > ./kong.sql
```

**数据导入**：

导入数据时首先创建数据库再用psql导入：

**权限问题，设置用户为超级用户**

```bash
$ createdb kong
$ psql -d kong -U kong -f kong.sql   # sql 文件在当前路径下 
$ psql -d databaename(数据库名) -U user(用户名) -f < 路径/文件名.sql # sql文件不在当前路径下
$ su postgres   #切换到psql用户下
$ psql -d wangye -U postgres -f system_calls.sql   # sql 文件在当前路径下
```

### 删除数据库

```bash
DROP DATABASE kong;
```

- 报错  ERROR:  database "kong" is being accessed by other user

### 删除模式(schema)

删除一个为空的模式（其中的所有对象已经被删除）：

```
DROP SCHEMA konga;
```

删除一个模式以及其中包含的所有对象：

```
DROP SCHEMA konga CASCADE;
```

### pgAdmin操作

数据的导出：
    在库名上右击-->backup-->ok，即将数据保存到.backup文件中。

数据的导入：
    在库名上右击-->restore-->注意填写.backup文件的路径不能有空格-->ok

---

## 几个简单命令

(1)列出所有的数据库

```
mysql: show databases
psql: \l或\list
```

(2)切换数据库

```
mysql: use dbname
psql: \c dbname
```

(3)列出当前数据库下的数据表

```
mysql: show tables
psql: \d
```

(4)列出指定表的所有字段

```
mysql: show columns from table name
psql: \d tablename
```

(5)查看指定表的基本情况

```
mysql: describe tablename
psql: \d+ tablename
```

(6)退出登录

```
mysql: quit 或者\q
psql:\q
```

(7)查看pgsl版本

```
pg_ctl --version
```

(8)命令行登陆数据库

```
psql -h 192.168.2.125 -p 5432 <dbname> <username>
```

(9)修改密码

```
psql登陆
然后， \password postgres
```

## 参考

**PostgreSQL 设置允许访问IP**
https://blog.csdn.net/wlchn/article/details/78915813

**postgresql数据库用户名密码验证失败**
https://blog.csdn.net/pg_hgdb/article/details/78805463

**PostgreSQL的访问控制（pg_hba.conf）**
https://my.oschina.net/liuyuanyuangogo/blog/497239

**Postgresql 远程连接配置**
https://www.cnblogs.com/3Tai/p/4935303.html

**PostgreSQL远程连接配置管理/账号密码分配**

https://yq.aliyun.com/articles/599287

**Postgres password authentication fails**
https://stackoverflow.com/questions/14564644/postgres-password-authentication-fails?rq=1
https://stackoverflow.com/questions/18664074/getting-error-peer-authentication-failed-for-user-postgres-when-trying-to-ge/26735105#26735105