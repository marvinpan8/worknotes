# SeaweedFS 集群部署

- **版本：4.40**
- wiki: https://github.com/seaweedfs/seaweedfs/wiki
- 存储比较：https://zhuanlan.zhihu.com/p/2001768773124899011

### 网络与端口

| 服务              | HTTP端口 | gRPC端口 |
| ----------------- | -------- | -------- |
| **Master**        | **9333** | 19333    |
| **Vol1**        | **8080** | 18080    |
| **Vol2**        | **8081** | 18081    |
| **Filer**         | **8888/8889(只读)** | 18888    |
| **S3** | **8333/8181(iceberg)** |    18333      |
| **prometheus** | **9327/9328/9329/9330** |          |

## ■■■ 一、硬件配置

- **因20T大容量磁盘，故下载 full large disk 最新版本** [linux_amd64_full_large_disk.tar.gz](https://github.com/seaweedfs/seaweedfs/releases/download/4.18/linux_amd64_full_large_disk.tar.gz)

```properties
mkdir -p /k0s/weed/bin
cd /k0s/weed/bin
tar -zxvf linux_amd64_full_large_disk.tar.gz
rm -f linux_amd64_full_large_disk.tar.gz
sudo cp weed /usr/local/bin
# 版本号 4.18 full large disk version
weed version
# 创建 master目录
sudo mkdir -p /data/weed/master
sudo chown -R ekemp:ekemp /data/weed
```

### 1. 每台物理机磁盘挂载

```properties
# 查看磁盘设备
lsblk
#查看本节点的文件系统信息：
df -lh
#查看本节点的磁盘信息：
sudo fdisk -l
# 格式化（必须使用 XFS，生产环境最佳实践）
sudo mkfs.xfs -i size=512 -f /dev/sda
sudo mkfs.xfs -i size=512 -f /dev/sdb
# 创建挂载点
sudo mkdir -p /weed/vol1  /weed/vol2
# 设置权限
sudo chown -R ekemp:ekemp /weed/vol1 /weed/vol2
```

### 2. 挂载配置

- `noatime,nodiratime`：避免每次访问更新文件时间戳，减少磁盘写入
- `allocsize=1M`：预分配大块空间，适合顺序写入，减少碎片

```properties
sudo blkid /dev/sda /dev/sdb 
sudo vim /etc/fstab
# 编辑添加，使用以下挂载选项：
UUID=xxx /weed/vol1 xfs defaults,noatime,nodiratime,allocsize=1M 0 0
UUID=yyy /weed/vol2 xfs defaults,noatime,nodiratime,allocsize=1M 0 0

# 测试所有 fstab 条目是否正确
sudo mount -a
# 查看当前挂载选项
mount | grep "/weed/vol1"
mount | grep "/weed/vol2"
# 查看实际配置 isize=512 imaxpct=5 (最大允许5%,默认25%)
sudo xfs_info /weed/vol1
# 容量监控（更重要）
df -h /weed/vol1
# inode 监控： 总19亿，空闲多少
df -i /weed/vol1
# 碎片监控（长期运行后检查）
sudo xfs_db -c "frag" -r /dev/sda
```

### 3. 修改限制文件数

```properties
# 软限制 1024，由每个进程自己调大
ulimit -n
# 登录 shell 的 硬限制 1048576
ulimit -n -H
# systemctl 服务的 硬限制  524288
systemctl show --property=DefaultLimitNOFILE

# 1.修改 PAM 限制（让终端和脚本也生效）创建新文件，99是最后加载
sudo ls -l /etc/security/limits.d
sudo tee /etc/security/limits.d/99-nofile.conf << 'EOF'
* soft nofile 65535
* hard nofile 524288
root soft nofile 65535
root hard nofile 524288
EOF
# 2. 重新登录（exit 后重新 ssh）
# 3. 验证
ulimit -n
# 应输出 65535
```



## ■■■ 二、 Master 启动命令

**在 101、102、103上执行**（3台Master高可用）：

```properties
# 创建模板配置文件
weed scaffold -config=master -output=/k0s/weed/default/
# 修改默认配置
sudo cp /k0s/weed/config/master.toml /etc/seaweedfs/

sudo vim /k0s/weed/options/master.conf
# 启动参数
sudo tee /k0s/weed/options/master.conf << EOF
ip=10.10.10.101
port=9333
peers=10.10.10.101:9333,10.10.10.102:9333,10.10.10.103:9333
defaultReplication=011
volumeSizeLimitMB=51200
garbageThreshold=0.15
mdir=/data/weed/master
metricsPort=9327
heartbeatInterval=300ms
electionTimeout=10s
resumeState=true
volumePreallocate=true
raftHashicorp=true
raftBootstrap=true
EOF
```

### 关键参数说明

| 参数                   | 值                    | 说明                                                         |
| ---------------------- | --------------------- | ------------------------------------------------------------ |
| **defaultReplication** | `011`                 | 2机架各保存1份，不同服务器保存3份副本                        |
| **volumeSizeLimitMB**  | `51200`               | 增大到50GB，减少GC频率，对HDD友好                            |
| **volumePreallocate**  | 启用                  | 预分配磁盘空间，减少碎片，提升顺序写入性能                   |
| **garbageThreshold**   | `0.15`                | **已删除数据占比超过阈值时，触发 Compact,并配合volume -compactionMBps 限速** |
| peers                  | 3个Master地址         | 形成Raft集群，实现高可用                                     |
| mdir                   |                       | 元数据存储目录                                               |
| metricsPort            | 9327                  | prometheus 监控端口                                          |
| master.metrics.address | http://localhost:9091 | Prometheus gateway address                                   |

### weed-master.service

```properties
sudo vim /etc/systemd/system/weed-master.service
# 创建 weed-master.service
sudo tee /etc/systemd/system/weed-master.service << EOF
[Unit]
Description=SeaweedFS Master Server
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=root
Group=root

ExecStart=/usr/local/bin/weed master -options=/k0s/weed/options/master.conf
WorkingDirectory=/opt/seaweedfs
LimitNOFILE=65535

Restart=on-failure
RestartSec=10s
TimeoutStopSec=30s

SyslogIdentifier=weed-master
StandardOutput=journal
StandardError=journal

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

#### 验证
```properties
sudo systemctl daemon-reload && sudo systemctl start weed-master
sudo systemctl enable weed-master
sudo systemctl is-enabled weed-master 
sudo systemctl stop weed-master
# 查看日志文件
sudo systemctl status weed-master
journalctl -f -u weed-master
tail -f -n 500 /var/log/syslog
sudo netstat -tunlp|grep 9333

# 集群验证
curl http://10.10.10.102:9333/cluster/status?pretty=y
# 查看集群容量和 Volume 分布情况
curl http://10.10.10.102:9333/dir/status?pretty=y
# 查看所有 Volume 的详细信息（ID、大小、副本策略等）
curl http://10.10.10.102:9333/vol/status?pretty=y

# weed shell 查看
weed shell -master=10.10.10.102:9333
# 检查集群网络连通性
cluster.check 
# 检查集群进程状态
cluster.ps
# 列出集群volume server
volume.list   
```



## ■■■ 三、Volume 启动

- 所有节点

```properties
sudo vim /k0s/weed/options/vol1.conf
# vol1 启动参数
sudo tee /k0s/weed/options/vol1.conf << EOF
ip=10.10.10.105
rack=rackB
dir=/weed/vol1
dir.idx=/data/weed/vol1
port=8080
port.grpc=18080
metricsPort=9328
dataCenter=dc1
master=10.10.10.101:9333,10.10.10.102:9333,10.10.10.103:9333
max=2400
disk=hdd
minFreeSpace=5
whiteList=10.10.10.0/24,10.10.20.0/24,10.244.0.0/16
index=leveldbMedium
index.leveldbTimeout=168
compactionMBps=20
maintenanceMBps=30
readBufferSizeMB=16
EOF

# vol2 启动参数
sudo tee /k0s/weed/options/vol2.conf << EOF
ip=10.10.10.105
rack=rackB
dir=/weed/vol2
dir.idx=/data/weed/vol2
port=8081
port.grpc=18081
metricsPort=9329
dataCenter=dc1
master=10.10.10.101:9333,10.10.10.102:9333,10.10.10.103:9333
max=2400
disk=hdd
minFreeSpace=5
whiteList=10.10.10.0/24,10.10.20.0/24,10.244.0.0/16
index=leveldbMedium
index.leveldbTimeout=168
compactionMBps=20
maintenanceMBps=30
readBufferSizeMB=16
EOF
```

### 关键参数说明

| 参数                 | 值                | 说明                                                 |
| -------------------- | ----------------- | ---------------------------------------------------- |
| dir                  | /weed/vol1        | 数据目录             |
| dir.idx                  | /data/weed/vol1 | 索引目录         |
| port                  | 8080 | 默认port         |
| port.grpc                  | port + 10000 | grpc port        |
| metricsPort                 | **9328** | 监控端口        |
| **max**              | **2400**          | 0表示每个目录不限Volume数量，master自动管理          |
| **index**            | **leveldbMedium** | **必须**！将索引放在磁盘而非内存，对机械硬盘至关重要 |
| **index.leveldbTimeout** | **168**（小时）=7天 | LevelDB空闲超时（小时），超时后卸载索引              |
| **maintenanceMBps**  | **30**            | 限制维护操作（修复/均衡）IO速率                      |
| **compactionMBps**   | **20（HDD推荐）** | 限制压缩速度，避免占满HDD带宽                        |
| **readBufferSizeMB** | **16**          | 读缓冲区大小（MB），机械盘可调高                     |
| **minFreeSpace** | **5**          | 磁盘剩余空间%，少于该值时整个Volume Server变为只读                  |
| **whiteList** | **10.10.10.0/24,10.244.0.0/16** | 白名单                  |


### weed-volume.service

```properties
sudo vim /etc/systemd/system/weed-vol1.service
# 创建 weed-vol1.service
sudo tee /etc/systemd/system/weed-vol1.service << EOF
[Unit]
Description=SeaweedFS Vol1 Server
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=root
Group=root

ExecStart=/usr/local/bin/weed volume -options=/k0s/weed/options/vol1.conf
WorkingDirectory=/opt/seaweedfs
LimitNOFILE=65535

Restart=on-failure
RestartSec=10s
TimeoutStopSec=30s

SyslogIdentifier=weed-vol1
StandardOutput=journal
StandardError=journal

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

# 创建 weed-vol2.service
sudo tee /etc/systemd/system/weed-vol2.service << EOF
[Unit]
Description=SeaweedFS Vol2 Server
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=root
Group=root

ExecStart=/usr/local/bin/weed volume -options=/k0s/weed/options/vol2.conf
WorkingDirectory=/opt/seaweedfs
LimitNOFILE=65535

Restart=on-failure
RestartSec=10s
TimeoutStopSec=30s

SyslogIdentifier=weed-vol2
StandardOutput=journal
StandardError=journal

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

#### 验证
```properties
sudo systemctl daemon-reload 
sudo systemctl start weed-vol1 weed-vol2

sudo systemctl enable weed-vol1 weed-vol2
sudo systemctl is-enabled weed-vol1 weed-vol2
sudo systemctl stop weed-vol1 weed-vol2
# 查看日志文件
sudo systemctl status weed-vol1 weed-vol2
journalctl -f -u weed-vol1
tail -f -n 500 /var/log/syslog
sudo netstat -tunlp|grep 9333
# 检查Volume Server状态
curl "http://10.10.10.103:8080/status?pretty=y"

# 验证集群
curl http://10.10.10.102:9333/cluster/status?pretty=y
# 查看集群容量和 Volume 分布情况
curl http://10.10.10.102:9333/dir/status?pretty=y
# 查看所有 Volume 的详细信息（ID、大小、副本策略等）
curl http://10.10.10.102:9333/vol/status?pretty=y


# weed shell 查看
weed shell -master=10.10.10.102:9333
# Volume 原生 API 上传 下载，不是 filer API 
weed upload -h
weed download -h
```



## ■■■ 四、Filer + S3 启动

**【201  202  203】  **（2个Filer即可实现高可用）

- **独立节点部署，不与k8s节点部署在一起，以免挂载PVC异常**

### 创建 MySQL 数据库表

```properties
# 创建数据库
CREATE DATABASE seaweedfs_filer CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;

# 创建用户
CREATE USER 'seaweedfs'@'10.10.20.20_' IDENTIFIED BY 'yNh2fD5HnRmUT4xg';
GRANT ALL PRIVILEGES ON seaweedfs_filer.* TO 'seaweedfs'@'10.10.20.20_';
FLUSH PRIVILEGES;
```

### 创建 `filer.toml` 配置数据库文件

```properties
# 创建模板配置文件
weed scaffold -config=filer -output=/k0s/weed/default/
# 修改默认配置
sudo cp /k0s/weed/config/filer.toml /etc/seaweedfs/
# 检查数据库
sudo vim /etc/seaweedfs/filer.toml 
```

### 创建 s3.json

```properties
sudo tee /etc/seaweedfs/s3.json << EOF
{
  "identities": [
    {
      "name": "admin",
      "credentials": [
        {
          "accessKey": "fe550f06-7bcc-4d16-a584-55b05a5f2403",
          "secretKey": "aa69b568-e38f-4e76-9c74-c4af2a7d8f65"
        }
      ],
      "actions": ["Admin", "Read", "Write", "List"]
    }
  ]
}
EOF
```

- Admin: 管理员权限
- Read: 读取权限
- Write: 写入权限
- List: 列表权限
- Tagging: 标签权限

### filer.conf

```properties
sudo vim /k0s/weed/options/filer.conf
# 启动参数
sudo tee /k0s/weed/options/filer.conf << EOF
ip=10.10.10.103
rack=rackA
port=8888
port.readonly=8889
master=10.10.10.101:9333,10.10.10.102:9333,10.10.10.103:9333
metricsPort=9330
defaultReplicaPlacement=011
dataCenter=dc1
disk=hdd
encryptVolumeData=true
downloadMaxMBps=50
s3=true
s3.port=8333
s3.port.iceberg=8181
s3.config=/etc/seaweedfs/s3.json
s3.allowDeleteBucketNotEmpty=false
s3.iam=true
s3.iam.readOnly=false
EOF
```

### 关键参数说明

| 参数                 | 值                | 说明                                                 |
| -------------------- | ----------------- | ---------------------------------------------------- |
| **defaultReplicaPlacement**  | **011**           | 必须设为 011，与 Master 配置保持一致                      |
| **encryptVolumeData**   | **true** | 加密 Volume 数据                        |
| **s3.encryptVolumeData** | **false**     | 不加密，与 filer API 一致 |
| **s3.allowDeleteBucketNotEmpty** | **false**          | 允许删除非空 Bucket                    |
| **s3.port.iceberg** | **8181（默认）**  | iceberg端口（0 是禁用）            |
| **downloadMaxMBps** | **50** | 单请求下载限速（MB/s）                  |

### weed-filer.service

- mkdir /opt/seaweedfs

```properties
sudo vim /etc/systemd/system/weed-filer.service
# 创建 weed-filer.service
sudo tee /etc/systemd/system/weed-filer.service << EOF
[Unit]
Description=SeaweedFS Filer Server
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=root
Group=root

ExecStart=/usr/local/bin/weed filer -options=/k0s/weed/options/filer.conf
WorkingDirectory=/opt/seaweedfs
LimitNOFILE=65535

Restart=on-failure
RestartSec=10s
TimeoutStopSec=30s

SyslogIdentifier=weed-filer
StandardOutput=journal
StandardError=journal

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

### 验证
```properties
# 再次检查 数据库配置文件
sudo cat /etc/seaweedfs/filer.toml 
# 启动
sudo systemctl enable weed-filer
sudo systemctl disable weed-filer
sudo systemctl is-enabled weed-filer

sudo systemctl daemon-reload 
sudo systemctl start weed-filer
sudo systemctl stop weed-filer
# 查看日志文件
sudo systemctl status weed-filer
journalctl -f -u weed-filer
tail -f -n 500 /var/log/syslog
sudo netstat -tunlp|grep weed

# 验证集群
curl http://10.10.10.102:9333/cluster/status?pretty=y
# 查看集群容量和 Volume 分布情况
curl http://10.10.10.102:9333/dir/status?pretty=y
# 查看所有 Volume 的详细信息（ID、大小、副本策略等）
curl http://10.10.10.102:9333/vol/status?pretty=y
# 检查 单个 Volume 状态
curl http://10.10.10.103:8080/status?pretty=y

# 测试上传
curl "http://10.10.10.102:9333/dir/assign?replication=011"
# 返回 {"fid":"3,016e37c6e6","url":"192.168.1.101:8080"}
curl -X PUT -F "file=@test.txt" "http://192.168.1.101:8080/3,016e37c6e6"
# 监控磁盘 IO
iostat -x 5
# 观察 %util 和 await
# 如果 %util < 70%，可适当调高,ompactionMBps=80

# weed shell 查看
weed shell -master=10.10.10.101:9333
# 检查集群网络连通性
> cluster.check 
# 检查集群进程状态
> cluster.ps
# 列出卷
> volume.list

# 每周执行数据完整性检查
> volume.fsck -collection=all
# 先模拟运行，找出Filer元数据中已经不存在的文件条目
> volume.fsck
# 如果确认找到的孤儿数据都需要删除，执行清理
> volume.fsck -reallyDeleteFromVolume

# 手动垃圾回收,物理数据清理
> volume.vacuum
> volume.vacuum -volumeId XXX
# 删除空卷
> volume.deleteEmpty -apply

# 查看目录
> fs.ls -l -a /topics
# 创建 匿名用户，指定 s3桶
> s3.bucket.access -name test-bucket -user anonymous -access Read,List
# 查看匿名用户权限
> s3.bucket.access -name test-bucket -user anonymous
# 删除 匿名用户，指定 s3桶
> s3.bucket.access -name test-bucket -user anonymous -access none
```

### MySQL  filemeta 查看

```properties
# 哪些目录占用了大量空间?
SELECT directory, COUNT(*) as file_count FROM filemeta GROUP BY directory ORDER BY file_count DESC LIMIT 20;
```





### 测试S3

```properties
sudo snap install aws-cli --classic
aws --version
# 配置 AWS CLI 使用测试凭证
aws configure list
aws configure
-------------------------
AWS Access Key ID [None]: fe550f06-7bcc-4d16-a584-55b05a5f2403
AWS Secret Access Key [None]: aa69b568-e38f-4e76-9c74-c4af2a7d8f65
Default region name [None]: us-east-1
Default output format [None]: json

# 测试访问（不再需要 --no-sign-request）
aws --endpoint-url http://10.10.10.66:9333 s3 ls
# 创建桶
aws --endpoint-url http://10.10.20.201:8333 s3 mb s3://loki-data
# 上传文件
echo "test" > test.txt
aws --endpoint-url http://10.10.20.201:8333 s3 cp test.txt s3://loki-data/
# 列出文件
aws --endpoint-url http://10.10.20.201:8333 s3 ls s3://loki-data/
# 查看 s3 元数据
aws --endpoint-url http://10.10.20.201:8333 s3api head-object --bucket test-bucket --key test.txt

# 清理文件，查看 mysql，删除记录了
aws --endpoint-url http://10.10.20.201:8333 s3 rm s3://loki-data/test.txt
# 清理文件，查看 mysql，删除表了
aws --endpoint-url http://10.10.20.201:8333 s3 rb s3://loki-data
```

### ■■■ S3 高可用查看另一篇【Haproxy-keepalived部署】

###  查看实际存储磁盘

Fid 由三个部分组成 【VolumeId, NeedleId, Cookie】

- VolumeId: 14 32bit 存储的物理卷的Id
- NeedleId: f4 64bit 全局唯一NeedleId，每个存储的文件都不一样(除了互为备份的)。
- Cookie: 187c8fdb 32bit Cookie值，为了安全起见，防止恶意攻击。

```properties
# 1. 查询 filer 元数据， 查看 fid.volume_id = 7; file_id = 7,1470f5a1a4
curl -H "Accept: application/json" "http://10.10.10.105:8888/buckets/test-bucket/test.txt?metadata=true&pretty=y"
# 2. 向 Master 查询 volumeId = 7 所在Volume 即 物理磁盘位置 102 102 104
curl "http://10.10.10.101:9333/dir/lookup?volumeId=7&pretty=y"
# 3. 根据以上所在 Volume 查看Collection 和 FileCount 
curl "http://10.10.10.102:8080/status?pretty=y"
{
      "Id": 7,
      "Size": 27,
      "ReplicaPlacement": {
        "node": 1,
        "rack": 1
      },
      "Ttl": {
        "Count": 0,
        "Unit": 0
      },
      "DiskType": "",
      "DiskId": 0,
      "Collection": "test-bucket",  # 桶名称
      "Version": 3,
      "FileCount": 1,    # 文件数量
      "DeleteCount": 0,
      "DeletedByteCount": 0,
      "ReadOnly": false,
      "CompactRevision": 0,
      "ModifiedAtSecond": 0,
      "RemoteStorageName": "",
      "RemoteStorageKey": ""
}   
# 3. 直接通过 filer API 访问文件
curl -H "Accept: application/json" http://10.10.10.102:8081/7,1470f5a1a4
curl -H "Accept: application/json" http://10.10.10.102:8081/7,169685fe78

# 都在  test-bucket_7.dat 中
sudo ls -l /weed/vol1
sudo ls -l /weed/vol2
```



## ■■■ 五、Admin UI

- **10.10.20. 201 单节点**
- **-master：master地址，多个用逗号隔开**
- **-dataDir: Admin UI 自身数据目录**
- **-adminPassword: 管理员（默认admin）密码**
### weed-admin.service

```properties
sudo mkdir /opt/seaweedfs
sudo mkdir -p /data/weed/admin
sudo vim /etc/systemd/system/weed-admin.service
# 创建 weed-admin.service
sudo tee /etc/systemd/system/weed-admin.service << EOF
[Unit]
Description=SeaweedFS admin Server
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/seaweedfs/seaweedfs/wiki

[Service]
Type=simple
User=root
Group=root

ExecStart=/usr/local/bin/weed admin -master="10.10.10.101:9333,10.10.10.102:9333,10.10.10.103:9333" -dataDir="/data/weed/admin" -adminPassword="Ekemp@2026"
WorkingDirectory=/opt/seaweedfs
LimitNOFILE=65535

Restart=on-failure
RestartSec=10s
TimeoutStopSec=30s

SyslogIdentifier=weed-admin
StandardOutput=journal
StandardError=journal

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

### 启动验证
```properties
sudo systemctl daemon-reload 
sudo systemctl start weed-admin

sudo systemctl enable weed-admin
sudo systemctl stop weed-admin
# 查看日志文件
sudo systemctl is-enabled weed-admin
sudo systemctl status weed-admin
journalctl -f -u weed-admin
tail -f -n 500 /var/log/syslog
sudo netstat -tunlp|grep weed
```

### 登录验证

```properties
# 用户名默认 admin / Ekemp@2026
http://10.10.20.201:23646
# 加速地址
http://8.218.51.188:50015
```




## ■■■ 六、删除卸载

1. **停止所有服务**：停止所有机器上的 Master、Volume Server、Filer 服务。
```properties
sudo systemctl stop weed-filer
sudo systemctl stop weed-vol1 weed-vol2
sudo systemctl stop weed-master
sudo systemctl stop weed-admin
```

2. **清理数据目录**：

```properties
# Master：删除 -mdir 参数指定的目录下的所有内容。
sudo rm -rf /data/weed/master/*
# Volume Server：删除 -dir 参数指定的所有目录下的内容。
sudo rm -rf /weed/vol1/* /weed/vol2/*
# MySQL：如需彻底重置，可清理 Filer 相关的数据库表。
DROP DATABASE seaweedfs_filer;
```



## ■■■ 七、升级部署

```properties
# 停止所有服务
sudo mv weed /usr/local/bin/
# 启动所有服务
```

