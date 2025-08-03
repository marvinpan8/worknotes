# Mongo7 分片模式集群安装

# 规划及部署(1分片+3副本)

参考：https://developer.aliyun.com/article/1345792

结合我们的生产需求，本次详细整理了最新版本 MonogoDB 7.0 集群的规划及部署过程，具有较大的参考价值，基本可照搬使用。适应数据规模为T级的场景，由于设计了分片支撑，后续如有大数据量需求，可分片横向扩展。

# ■■■ 分片集群规划

## Configure hostname、hosts file、ip address

vim /etc/hosts

```properties
172.17.0.30 node1
172.17.0.31 node2
172.17.0.32 node3
```

注：规划、实施、运维均采用host解析的方式判定各个节点，因此需确保该配置文件需正确解析node1、node2、node3.

## ■ 节点的角色及端口分配

```elm
┌──── 名称────┌────node1────┬────node2────┬────node3────┬ 实际端口─┬ 默认端口─┐
│  路由节点    │mongos server│mongos server│mongos server│  20000 │  27017  │
├──────────── ├─────────────┼─────────────┼─────────────┼────────┤─────────┤
│  配置节点    │config server│config server│config server│  21000 │  27018  │
│             │(Primary)    │(Secondary)  │(Secondary)  │        │         │
├─────────────├─────────────┼─────────────┼─────────────┼────────┤─────────┤
│  分片节点1   │shard1 server│shard1 server│shard1 server│  27001 │  27019  │
│             │(Primary)    │(Secondary)  │(Secondary)  │        │         │
├─────────────├─────────────┼─────────────┼─────────────┼────────┤─────────┤
│  分片节点2   │shard2 server│shard2 server│shard2 server│  27002 │  27019  │
│             │(Secondary)  │(Primary)    │(Secondary)  │        │         │
├─────────────├─────────────┼─────────────┼─────────────┼────────┤─────────┤
│  分片节点3   │shard3 server│shard3 server│shard3 server│  27003 │  27019  │
│             │(Secondary)  │(Secondary)  │(Primary  )  │        │         │
└─────────────└─────────────┴─────────────┴─────────────┴────────┴─────────┘
```

# ■■■ 环境准备

## 依赖包

```properties
yum install -y libcurl openssl xz-libs
```

## 用户及用户组

```properties
groupadd mongod
groupadd mongodb
useradd -g mongod -G mongodb mongod
echo "passwd"|passwd mongod --stdin
```

## ■ mongodb 下载安装

官方下载：https://www.mongodb.com/try/download

```bash
#20250709 最新版本，选择合适的平台介质
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-7.0.21.tgz

mkdir /usr/local/mongodb
KDR=/usr/local/mongodb
cd ${KDR}
TGZ=mongodb-linux-x86_64-rhel70-7.0.21

#cp_unzip_chown_ln:
cp ~/${TGZ}.tgz .
tar -zxvf ${TGZ}.tgz && rm -f ${TGZ}.tgz 
chown -R mongod:mongod ${TGZ}
ln -s ${KDR}/${TGZ}/bin/* /usr/local/bin/
```

## ■ 客户端database-tools 下载安装(只装一台)

6.0版本开始，将数据库相关的工具单独管理，以利于实时升级、发布
wget  https://fastdl.mongodb.org/tools/db/mongodb-database-tools-rhel70-x86_64-100.12.2.tgz

```properties
KDR=/usr/local/mongodb
cd ${KDR}
TGZ=mongodb-database-tools-rhel70-x86_64-100.12.2
【后续步骤同上】
#cp_unzip_chown_ln:
cp ~/${TGZ}.tgz .
tar -zxvf ${TGZ}.tgz && rm -f ${TGZ}.tgz 
chown -R mongod:mongod ${TGZ}
ln -s ${KDR}/${TGZ}/bin/* /usr/local/bin/
```

## ■ 客户端mongosh 下载安装(只装一台)

<https://downloads.mongodb.com/compass/mongosh-2.5.5-linux-x64.tgz>

```properties
KDR=/usr/local/mongodb
cd ${KDR}
TGZ=mongosh-2.5.5-linux-x64
【后续步骤同上】
#cp_unzip_chown_ln:
cp ~/${TGZ}.tgz .
tar -zxvf ${TGZ}.tgz && rm -f ${TGZ}.tgz 
chown -R mongod:mongod ${TGZ}
ln -s ${KDR}/${TGZ}/bin/* /usr/local/bin/
```

---

## ■ 3个节点创建mongodb数据库文件目录

```properties
MongoDir=/data/mongodb
mkdir -p ${MongoDir}
chown -R mongod:mongod ${MongoDir}

cat >> /etc/profile << EOF
export MongoDir=${MongoDir}
EOF

source /etc/profile
```

## ■ 以下均用mongod用户操作

```properties
su - mongod
echo ${MongoDir}
```

## ■ 3个节点均建立6个目录：conf、mongos、config、shard1、shard2、shard3

```properties
mkdir -p ${MongoDir}/conf
mkdir -p ${MongoDir}/mongos/log
mkdir -p ${MongoDir}/config/data
mkdir -p ${MongoDir}/config/log
mkdir -p ${MongoDir}/shard1/data
mkdir -p ${MongoDir}/shard1/log
mkdir -p ${MongoDir}/shard2/data
mkdir -p ${MongoDir}/shard2/log
mkdir -p ${MongoDir}/shard3/data
mkdir -p ${MongoDir}/shard3/log

# 删除(忽略)
cd /data/mongodb
rm -rf config mongos shard1 shard2 shard3
```

## ■ 检查目录结构

```elm
tree ${MongoDir} -L 2 --dirsfirst
-------------------------------------------
├── conf
│   ├── config.conf
│   ├── mongos.conf
│   ├── shard1.conf
│   ├── shard2.conf  # 暂时没有
│   └── shard3.conf  # 暂时没有
├── config
│   ├── data
│   └── log
├── mongos
│   └── log
├── shard1
│   ├── data
│   └── log
├── shard2
│   ├── data
│   └── log
└── shard3
    ├── data
    └── log
```





# ■■■ config server

mongodb3.4以后要求配置服务器也创建副本集，不然集群搭建不成功

## ■ 配置文件3个节点

参考：

https://www.mongodb.com/zh-cn/docs/manual/reference/configuration-options/

https://www.mongodb.com/zh-cn/docs/v7.0/reference/parameters/

```properties
cat > ${MongoDir}/conf/config.conf << EOF
processManagement:
  # fork: true
  pidFilePath: ${MongoDir}/config/log/configsvr.pid
net:
  bindIp: 0.0.0.0
  port: 21000
  maxIncomingConnections: 20000
storage:
  dbPath: ${MongoDir}/config/data
  directoryPerDB: true
  syncPeriodSecs: 120
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2
      directoryForIndexes: true
      journalCompressor: zlib
systemLog:
  destination: file
  path: ${MongoDir}/config/log/configsvr.log
  logAppend: true
sharding:
  clusterRole: configsvr
replication:
  replSetName: configs
setParameter:
  authenticationMechanisms: SCRAM-SHA-256
  connPoolMaxConnsPerHost: 20000
EOF
```

##  ■ 3个节点配置系统自启动

#### root 登陆

```properties
cat << EOF | tee /etc/systemd/system/mongodConfig.service
[Unit]
Description=MongoDB Config Server
Documentation=https://docs.mongodb.org/manual
After=network-online.target
Wants=network-online.target

[Service]
User=mongod
Group=mongod
ExecStart=/usr/local/bin/mongod -f /data/mongodb/conf/config.conf
ExecStop=/usr/local/bin/mongod -f /data/mongodb/conf/config.conf --shutdown
Restart=on-failure
LimitFSIZE=infinity
LimitCPU=infinity
LimitAS=infinity
LimitNOFILE=64000
LimitNPROC=64000
LimitMEMLOCK=infinity
TasksMax=infinity
TasksAccounting=false


[Install]
WantedBy=multi-user.target
EOF
```
### 启动

```properties
systemctl daemon-reload && systemctl restart mongodConfig
-----------------------------------------
systemctl enable mongodConfig 
systemctl status mongodConfig
journalctl -f -u mongodConfig
tail -f -n 500 /data/mongodb/config/log/configsvr.log
ps -ef | grep mongo
netstat -tunlp|grep mongod

systemctl stop mongodConfig
```

## ■ 客户端登录任意一台配置服务器，初始化配置副本集

```properties
mongosh node1:21000

# 定义config变量：
config = {_id: "configs", members: [
  {_id: 0, host: "node1:21000"},
  {_id: 1, host: "node2:21000"},
  {_id: 2, host: "node3:21000"} ]
}

# 其中，_id: "configs"应与配置文件中的配置一致，"members" 中的 "host" 为三个节点的 ip 和 port
# 初始化副本集：
rs.initiate(config)

# 查看此时状态：
rs.status()
rs.isMaster()
rs.conf();
```





# ■■■ 创建帐号和认证

> 客户端mongosh，通过localhost或127.0.0.1登录任意一个mongos路由，可以执行创建操作
> 提示：此时相当于一个后门，只能在 admin 下添加用户
> 提示：通过mongos添加的账号信息，只会保存到配置节点的服务中，具体的数据节点不保存账号信息，因此分片中的账号信息不涉及到同步问题
> 建议：先创建超管用户和普通用户，然后再开启安全配置

## ■ 创建管理员帐号：

```properties
use admin
db.createUser({user: "admin", pwd: "passwd!2#", roles: ["root"]})
```

---
# ■■■ 用户权限配置

对于搭建好的mongodb分片集群，为了安全，需启动安全认证，使用账号密码登录。
默认的mongodb是不设置认证的。只要ip和端口正确就能连接，这样是很不安全的。
mongodb官网声称，为了能保障mongodb的安全可以做以下几个步骤：

1. **使用新的端口，默认的27017端口如果一旦知道了ip就能连接上，不太安全**

2. **设置mongodb的网络环境，最好将mongodb部署到公司服务器内网，这样外网是访问不到的。公司内部访问使用vpn等**

3. **开启安全认证。认证要同时设置服务器之间的内部认证方式，同时要设置客户端连接到集群的账号密码认证方式**

以下详细描述如何配置安全认证。

## ■ node1 创建副本集认证的key文件

用openssl生成密码文件，然后使用chmod来更改文件权限，仅为文件所有者提供读取权限

```properties
cd ${MongoDir}/conf
openssl rand -out mongo.keyfile -base64 90
chmod 600 mongo.keyfile
ll mongo.keyfile
cat mongo.keyfile
-------------------------------------------

```

> 提示：所有副本集节点都必须要用同一份keyfile，一般是在一台机器上生成，然后拷贝到其他机器上，且必须有读的权限，否则将来会报错：
> permissions on ${MongoDir}/conf/mongo.keyfile are too open

## ■ node1 将修改后的配置文件和key文件拷贝到 node2、node3

```bash
scp ${MongoDir}/conf/mongo.keyfile node2:${MongoDir}/conf
scp ${MongoDir}/conf/mongo.keyfile node3:${MongoDir}/conf
```


## ■ 3个节点修改config.conf

先停止进程：  

```properties
systemctl stop mongodConfig
ps -ef | grep mongo
```

增加鉴权配置  vim config.conf

```properties
security:
   authorization: enabled
   keyFile: /data/mongodb/conf/mongo.keyfile
```

## ■ 重新启动3个节点的config程序

```properties
mongod -f ${MongoDir}/conf/config.conf --shutdown
-------------------------------------------------
Killing process with pid: 18559
-------------------------------------------------
ps -ef |grep mongo
mongod -f ${MongoDir}/conf/config.conf
```
## ■ 客户端登录测试

```properties
mongosh node1:21000

use admin
db.auth("admin", "passwd!2#")
show databases;
rs.status()
```





---

# ■■■ shard server

## ■ shard server1

【3个节点执行】
【注意】如果数据量并不大，分片需求不明显，可以先只创建shard server1，另外的分片2、分片3先不创建，后续根据实际需求可随时创建。

参考：

https://www.mongodb.com/zh-cn/docs/v7.0/reference/configuration-options/

https://www.mongodb.com/zh-cn/docs/v7.0/reference/parameters/

```properties
cat > ${MongoDir}/conf/shard1.conf << EOF
processManagement:
  # fork: true
  pidFilePath: ${MongoDir}/shard1/log/shard1.pid
net:
  bindIp: 0.0.0.0
  port: 27001
  maxIncomingConnections: 20000
storage:
  dbPath: ${MongoDir}/shard1/data
  directoryPerDB: true
  syncPeriodSecs: 120
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 5
      directoryForIndexes: true
      journalCompressor: zlib
systemLog:
  destination: file
  path: ${MongoDir}/shard1/log/shard1.log
  logAppend: true
sharding:
  clusterRole: shardsvr
replication:
  replSetName: shard1
setParameter:
  connPoolMaxConnsPerHost: 20000
  maxNumActiveUserIndexBuilds: 6
  authenticationMechanisms: SCRAM-SHA-256
security:
  authorization: enabled
  keyFile: ${MongoDir}/conf/mongo.keyfile
EOF
```
##  ■ 3个节点配置系统自启动

#### root 登陆

```properties
cat << EOF | tee /etc/systemd/system/mongodShard1.service
[Unit]
Description=MongoDB shard1 Server
Documentation=https://docs.mongodb.org/manual
After=network-online.target
Wants=network-online.target

[Service]
User=mongod
Group=mongod
ExecStart=/usr/local/bin/mongod -f /data/mongodb/conf/shard1.conf
ExecStop=/usr/local/bin/mongod -f /data/mongodb/conf/shard1.conf --shutdown
Restart=on-failure
LimitFSIZE=infinity
LimitCPU=infinity
LimitAS=infinity
LimitNOFILE=64000
LimitNPROC=64000
LimitMEMLOCK=infinity
TasksMax=infinity
TasksAccounting=false


[Install]
WantedBy=multi-user.target
EOF
```
### 启动

```properties
systemctl daemon-reload && systemctl restart mongodShard1
-----------------------------------------
systemctl enable mongodShard1 
systemctl status mongodShard1
journalctl -f -u mongodShard1
tail -f -n 500 /data/mongodb/shard1/log/shard1.log
ps -ef | grep mongo
netstat -tunlp|grep mongod
```


## ■ 登陆任意节点，初始化副本集：

注：初始化副本集的操作不能在仲裁节点上执行！在哪个节点初始化，则哪个节点默认是副本集的主节点。

```properties
mongosh --port 27001
# 使用admin数据库，定义副本集配置，
use admin
db.createUser({user: "admin", pwd: "passwd!2#", roles: ["root"]})
db.auth("admin", "passwd!2#")
#模式选择 P/S/S
config = {_id: "shard1", members: [
    {_id: 0, host: "node1:27001"},
    {_id: 1, host: "node2:27001"},
    {_id: 2, host: "node3:27001"}
  ]
}
#模式选择 P/S/A, 5节点时应用; "arbiterOnly":true 代表仲裁节点,会忽略该节点的数据存储功能（跳过）
config = {_id: "shard1", members: [
    {_id: 0, host: "node1:27001"},
    {_id: 1, host: "node2:27001"},
    {_id: 2, host: "node3:27001", arbiterOnly:true}
  ]
}
rs.initiate(config);
# db.auth("admin", "passwd!2#")
rs.status()
rs.isMaster()
rs.conf();
```

## ■ shard server2 【备用，暂不执行】

## ■ shard server3 【备用，暂不执行】





---

# ■■■ 路由节点

## ■ mongos server1

【3个节点执行】
【注意】如果数据量并不大，分片需求不明显，可以先只创建shard server1，另外的分片2、分片3先不创建，后续根据实际需求可随时创建。[`security.authorization`](https://www.mongodb.com/zh-cn/docs/v7.0/reference/configuration-options/#mongodb-setting-security.authorization) 设置仅适用于 [`mongod`](https://www.mongodb.com/zh-cn/docs/v7.0/reference/program/mongod/#mongodb-binary-bin.mongod)。

参考：

https://www.mongodb.com/zh-cn/docs/v7.0/reference/configuration-options/

https://www.mongodb.com/zh-cn/docs/v7.0/reference/parameters/

```properties
cat > ${MongoDir}/conf/mongos.conf << EOF
processManagement:
  # fork: true
  pidFilePath: ${MongoDir}/mongos/log/mongos.pid
net:
  bindIp: 0.0.0.0
  port: 20000
  maxIncomingConnections: 20000
systemLog:
  destination: file
  path: ${MongoDir}/mongos/log/mongos.log
  logAppend: true
replication:
   localPingThresholdMs: 15
sharding:
  configDB: configs/node1:21000,node2:21000,node3:21000
setParameter:
  connPoolMaxConnsPerHost: 20000
  authenticationMechanisms: SCRAM-SHA-256
security:
  keyFile: ${MongoDir}/conf/mongo.keyfile
EOF
```
##  ■ 3个节点配置系统自启动

#### root 登陆

```properties
cat << EOF | tee /etc/systemd/system/mongos.service
[Unit]
Description=MongoDB mongos Server
After=network-online.target
Wants=network-online.target

[Service]
User=mongod
Group=mongod
ExecStart=/usr/local/bin/mongos -f /data/mongodb/conf/mongos.conf
ExecStop=/bin/kill -TERM $MAINPID
RuntimeDirectory=/data/mongodb/mongos/
Restart=on-failure
LimitFSIZE=infinity
LimitCPU=infinity
LimitAS=infinity
LimitNOFILE=64000
LimitNPROC=64000
LimitMEMLOCK=infinity
TasksMax=infinity
TasksAccounting=false


[Install]
WantedBy=multi-user.target
EOF
```
### 启动

```properties
systemctl daemon-reload && systemctl enable mongos && systemctl start mongos
-----------------------------------------
systemctl status mongos
journalctl -f -u mongos
tail -f -n 500 /data/mongodb/mongos/log/mongos.log
ps -ef | grep mongos
netstat -tunlp|grep mongos

systemctl stop mongos
ll /tmp/mongodb-20000.sock
```




# ■■■ 测试

## ■ 用管理员帐号可查看整体的分片情况

```properties
mongosh node1:21000
use admin
db.auth("admin", "passwd!2#")
show dbs;
rs.status()
```

连接 mongos 服务器 输出 分片信息： sh.status()

MongoshInvalidInputError: [SHAPI-10003] This db does not have sharding enabled. Be sure you are connecting to a mongos from the shell and not to a mongod.

- **shards：分片的数据库**

```properties
shardingVersion
{ _id: 1, clusterId: ObjectId('686f2dfa2ae795f50f5df660') }
---
shards
[]
---
most recently active mongoses
'none'
---
autosplit
{ 'Currently enabled': 'yes' }
---
balancer
{
  'Failed balancer rounds in last 5 attempts': 0,
  'Currently running': 'unknown',
  'Currently enabled': 'yes',
  'Migration Results for the last 24 hours': 'No recent migrations'
}
---
shardedDataDistribution
undefined
---
databases
[
  {
    database: { _id: 'config', primary: 'config', partitioned: true },
    collections: {}
  }
]
```

## ■ 客户端连接多个mongos的标准格式

```properties
mongosh mongodb://node1:20000,node2:20000,node3:20000/admin?authSource=admin
mongosh mongodb://cx-user:cx-admin@172.17.0.222:30756/?directConnection=true
```

-----------

# 导出导入数据

- mongodump 和mongorestore 主要用于**【数据库】**级别的导出和导入，尽管也可以用于**【集合】**的操作。

- mongoexport 和mongoimport则主要用于**【集合】**的导出和导入。



## mongodump & mongorestore

参考： https://www.mongodb.com/zh-cn/docs/database-tools/mongodump/

参考： https://www.mongodb.com/zh-cn/docs/database-tools/mongorestore/

#### 导出数据库db1 到目录export_data_2023_1210:

- 加 -o 指定导出数据目录

```properties
mongodump --host 172.17.0.222 --port 30756 \
    --username cx-user --password cx-admin \
    --db db_usercenter \
    --authenticationDatabase admin \
    -o .
```
#### 导入数据库db1
```properties
cd /data/mongodb/tmp20250711
# 导入所有库
mongorestore --host node1 --port 20000 \
    --username admin --password 'passwd!2#' \
    --authenticationDatabase admin \
    ./
# 导入指定库和集合
mongorestore --host node1 --port 20000 \
    --username admin --password 'passwd!2#' \
    --authenticationDatabase admin \
    --db db_device_manage /data/mongodb/tmp20250711/db_device_manage/device.bson
```



## mongoexport & mongoimport

#### 导出集合amias_db数据库下的user_1

```properties
mongoexport -d amias_db \
    -c user_1 -o user_1.json \
    -u dba --port=27018 \
    --authenticationDatabase admin
```
#### 导入集合user_2
```properties
mongoimport -d amias_db \
    -c user_2 \
    -u dba --port=27018 \
    --authenticationDatabase admin \
    --file ./user_1.json
```
#### 当然也可以在导出时带条件:
```properties
mongoexport -d db2 \
    -c collection_example \
    -o collection_example.json \
    -u dba --port=27018 \
    --authenticationDatabase admin \
    -q '{"last_update":{"$gt": "2023-10-01T00:01:01.011454+00:00"}}'
```
#### 导出指定字段
```properties
mongoexport -d amias_db \
    -c user_2 \
    --type=csv \
    --fields=user_name \
    -o user_2_name.csv \
    -u dba --port=27018 \
    --authenticationDatabase admin
```

# db未分片处理

**此方法必须在mongos实例上运行。**

1. 确保所有分片服务器运行
确保所有的分片服务器（mongod实例）都在运行，并且已经加入到集群中。你可以使用以下命令查看集群成员：
```properties
# 查看 shards
sh.status()
```
3. 检查Config Server状态
Config Server是存储元数据的地方，包括分片信息。确保Config Server正在运行并且配置正确：
```properties
rs.conf()
```
4. 重新启用分片功能（如果需要）如果分片功能被禁用，你需要重新启用它：
```properties
use db_device_manage
sh.enableSharding("db_device_manage")

use db_foodSec
sh.enableSharding("db_foodSec")

use db_foodsample
sh.enableSharding("db_foodsample")

use db_usercenter
sh.enableSharding("db_usercenter")

# 对test.t集合以id列为shard key进行hashed sharding
sh.shardCollection("test.t",{id:"hashed"}) 
# 对test.t集合以id列为shard key进行ranged sharding，直接使用{id:1}方式指定即可，分片的chunk由mongos自主决定
sh.shardCollection("test.t",{id:1})
# 查看自动为id列创建了索引
db.t.getIndexes()
```
5. 添加分片（如果缺少）如果缺少分片，你可以添加一个新的分片：
```properties
sh.addShard("shard1/node1:27001,node2:27001,node3:27001")
# 以后添加（跳过）
sh.addShard("shard2/node1:27002,node2:27002,node3:27002")
sh.addShard("shard3/node1:27003,node2:27003,node3:27003")
```

##  ■ 创建用户

```properties
use admin
db.auth("admin", "passwd!2#")
db.getUsers()

db.createUser({user: "cx-user", pwd: "cx-admin", roles: [{ role: 'readWrite', db: 'db_device_manage' },{ role: 'readWrite', db: 'db_foodSec' },{ role: 'readWrite', db: 'db_foodsample' },{ role: 'readWrite', db: 'db_usercenter' }]})

db.auth("cx-user", "cx-admin")
```

## ■ 鉴权操作（忽略）
```properties
# 鉴权操作：
db.auth("admin", "passwd!2#")
# 新增授权
db.grantRolesToUser("<username>", [{ role: "dbAdmin", db: "<database>" }])
# 覆盖权限
db.updateUser("<username>", [{ role: "dbAdmin", db: "<database>" }])
```

## ■ 创建普通权限帐号（忽略）

```properties
use cloudconf
db.createUser({user: "mms", pwd: "passwd!2#", roles: ["readWrite"]})
# 新增授权
db.grantRolesToUser("mms", [{ role: "readWrite", db: "cloudconf" }])
db.grantRolesToUser("mms", [{ role: "dbOwner", db: "cloudconf" }])
db.auth("mms", "passwd!2#")
```

---------------

# mongostat 命令性能监控

```properties
mongostat -h node3 --port=20000 --username=admin --password='passwd!2#' --authenticationDatabase=admin --discover -n 300 2
```

参数说明:

- -h：指定监听的主机，分片集群模式下指定到一个mongos实例，也可以指定单个mongod，或者复制集的多个节点。
- --port：接入的端口，如果不提供则默认为27017。
- -u：接入用户名，等同于--username。
- -p：接入密码，等同于--password。
- --authenticationDatabase：鉴权数据库。
- --discover：启用自动发现，可展示集群中所有分片节点的状态。
- -n 300 2：表示输出300次，每次间隔2s。也可以不指定“-n 300”，此时会一直保持输出。



| 指标名 | 说明 |
| ---- | ---- |
| inserts | 每秒插入数 |
| query | 每秒查询数 |
| update | 每秒更新数 |
| delete | 每秒删除数 |
| getmore | 每秒getmore数 |
| command | 每秒命令数，涵盖了内部的一些操作 |
| **%dirty** | WiredTiger缓存中脏数据百分比 |
| %used  | WiredTiger 正在使用的缓存百分比 |
| flushes | WiredTiger执行CheckPoint的次数 |
| vsize | 虚拟内存使用量 |
| res | 物理内存使用量 |
| **qrw** | 客户端读写等待队列数量，高并发时，一般队列值会升高 |
| **arw** | 客户端读写活跃个数 |
| **netIn** | 网络接收数据量 |
| **netOut** | 网络发送数据量 |
| **conn** | 当前连接数 |
| set | 所属复制集名称 |
| **repl** | 复制节点状态（主节点/二级节点……) |
| time | 时间戳 |


mongostat需要关注的指标主要有如下几个：

- 插入、删除、修改、查询的速率是否产生较大波动，是否超出预期。
- qrw、arw：队列是否较高，若长时间大于0则说明此时读写速度较慢。
- conn：连接数是否太多。
- dirty：百分比是否较高，若持续高于10%则说明磁盘I/O存在瓶颈。
- netIn、netOut：是否超过网络带宽阈值。
- repl：状态是否异常，如PRI、SEC、RTR为正常，若出现REC等异常值则需要修复。

# mongotop 命令性能监控

```properties
mongotop -h node3 --port=21000 --username=admin --password='passwd!2#' --authenticationDatabase=admin
mongotop -h node3 --port=27001 --username=admin --password='passwd!2#' --authenticationDatabase=admin
```
输出

```
                                 ns    total    read    write    2025-07-10T19:40:59+08:00
                  admin.system.keys      0ms     0ms      0ms                             
                 admin.system.users      0ms     0ms      0ms                             
               admin.system.version      0ms     0ms      0ms                             
config.collection_critical_sections      0ms     0ms      0ms                             
    config.external_validation_keys      0ms     0ms      0ms                             
                    config.settings      0ms     0ms      0ms                             
           config.shard.collections      0ms     0ms      0ms                             
               config.shard.indexes      0ms     0ms      0ms                             
        config.shardMergeRecipients      0ms     0ms      0ms                             
            config.shardSplitDonors      0ms     0ms      0ms 
```



| 指标名 | 说明 |
| ---- | ---- |
| ns | 集合名称空间 |
| total | 花费在该集合上的时长 |
| read | 花费在该集合上的读操作时长 |
| write | 花费在该集合上的写操作时长 |

---

# 集群启动和关闭顺序

## 分片(Shard)环境中的启动和关闭

1. 启动 ：这个具体的参照分片的配置，启动的顺序是
  ```properties
config server -> 副本集/分片(shardX) -> mongos
  ```
2. 关闭： 因为mongos是分片架构最前端的入口，所以关闭顺序： 
  ```properties
mongos -> 副本集/分片(shardX) -> config server
  ```

## Cluster集群启停
### 停库：
```properties
# 1、如果有打开平衡的，记得先关闭平衡。
sh.stopBalancer()                    
sh.getBalancerState()
# 2、先停mongos，
db.getSiblingDB(“admin”).shutdownServer()
# 3、再停shard
db.getSiblingDB(“admin”).shutdownServer()
# 4、最后停configSer
db.getSiblingDB(“admin”).shutdownServer()
```

### 启库：
```properties
# 1、优先启动configSer
mongod -f /mongo-config.conf
# 2、其次分片：
mongod -f /mongo-shardx.conf
# 3、最后mongos
mongos -f /mongo-mongos.conf
# 4、如果之前有打开平衡，就打开，没有就忽略这两步
sh.startBalancer()                 
sh.getBalancerState()
```
# ■■■ 添加删除shard 节点（慎用忽略）

#### 节点停止之前先停止从节点可以避免数据不一致

```properties
# 增加节点
rs.add("node3:27001")
# 移除节点，会终止分片进程，慎用
rs.remove("node3:27001")

# 添加仲裁节点
rs.addArb("node3:27001")
# 移除仲裁节点,会终止分片进程，慎用
rs.remove("node3:27001")
```

