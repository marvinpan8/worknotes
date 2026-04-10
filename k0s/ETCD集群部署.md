# ETCD集群部署

#### ETCD版本：3.6.9

官网地址：https://github.com/etcd-io/etcd/tags

## 创建安装目录

```bash
sudo mkdir -p /k8s/etcd/{bin,cfg,ssl} 
# 创建数据目录
sudo mkdir -p /data/etcd/data /data/etcd/wal
```
## 解压安装文件

```properties
mkdir ~/etcd && cd ~/etcd
tar -xvf etcd-v3.6.9-linux-amd64.tar.gz
rm -f etcd-v3.6.9-linux-amd64.tar.gz 
cd etcd-v3.6.9-linux-amd64/
sudo cp etcd etcdctl /k8s/etcd/bin/

sudo cp /k8s/etcd/bin/etcdctl /usr/local/bin/
sudo cp /k8s/etcd/bin/etcd /usr/local/bin/

# 校验
etcd --version
etcdctl version
```
## etcd01

```properties
sudo tee /k8s/etcd/cfg/etcd.conf << EOF
name: 'etcd01'

data-dir: /data/etcd/data
wal-dir: /data/etcd/wal

heartbeat-interval: 250
election-timeout: 2000
quota-backend-bytes: 6442450944

listen-peer-urls: https://10.10.20.201:2380
listen-client-urls: https://10.10.20.201:2379,https://127.0.0.1:2379
initial-advertise-peer-urls: https://10.10.20.201:2380
advertise-client-urls: https://10.10.20.201:2379
initial-cluster: etcd01=https://10.10.20.201:2380,etcd02=https://10.10.20.202:2380,etcd03=https://10.10.20.203:2380

initial-cluster-token: 'etcd-cluster'
initial-cluster-state: 'new'

strict-reconfig-check: false
enable-pprof: true
proxy: 'off'

client-transport-security:
  cert-file: /k8s/etcd/ssl/server-cert.pem
  key-file: /k8s/etcd/ssl/server-key.pem
  trusted-ca-file: /k8s/etcd/ssl/ca.pem
  client-cert-auth: true

  auto-tls: false

peer-transport-security:
  cert-file: /k8s/etcd/ssl/peer-cert.pem
  key-file: /k8s/etcd/ssl/peer-key.pem
  trusted-ca-file: /k8s/etcd/ssl/ca.pem
  client-cert-auth: true

  auto-tls: false
  allowed-cn:
  allowed-hostname:

self-signed-cert-validity: 100
log-level: info
logger: zap
log-outputs: [stderr]
force-new-cluster: false
auto-compaction-mode: periodic
auto-compaction-retention: "1"

cipher-suites: [
  TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
]

tls-min-version: 'TLS1.2'
tls-max-version: 'TLS1.3'
EOF
```
## etcd02

```properties
sudo tee /k8s/etcd/cfg/etcd.conf << EOF
name: 'etcd02'

data-dir: /data/etcd/data
wal-dir: /data/etcd/wal

heartbeat-interval: 250
election-timeout: 2000
quota-backend-bytes: 6442450944

listen-peer-urls: https://10.10.20.202:2380
listen-client-urls: https://10.10.20.202:2379,https://127.0.0.1:2379
initial-advertise-peer-urls: https://10.10.20.202:2380
advertise-client-urls: https://10.10.20.202:2379
initial-cluster: etcd01=https://10.10.20.201:2380,etcd02=https://10.10.20.202:2380,etcd03=https://10.10.20.203:2380

initial-cluster-token: 'etcd-cluster'
initial-cluster-state: 'new'

strict-reconfig-check: false
enable-pprof: true
proxy: 'off'

client-transport-security:
  cert-file: /k8s/etcd/ssl/server-cert.pem
  key-file: /k8s/etcd/ssl/server-key.pem
  trusted-ca-file: /k8s/etcd/ssl/ca.pem
  client-cert-auth: true

  auto-tls: false

peer-transport-security:
  cert-file: /k8s/etcd/ssl/peer-cert.pem
  key-file: /k8s/etcd/ssl/peer-key.pem
  trusted-ca-file: /k8s/etcd/ssl/ca.pem
  client-cert-auth: true

  auto-tls: false
  allowed-cn:
  allowed-hostname:

self-signed-cert-validity: 100
log-level: info
logger: zap
log-outputs: [stderr]
force-new-cluster: false
auto-compaction-mode: periodic
auto-compaction-retention: "1"

cipher-suites: [
  TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
]

tls-min-version: 'TLS1.2'
tls-max-version: 'TLS1.3'
EOF
```
## etcd03

```properties
sudo tee /k8s/etcd/cfg/etcd.conf << EOF
name: 'etcd03'

data-dir: /data/etcd/data
wal-dir: /data/etcd/wal

heartbeat-interval: 250
election-timeout: 2000
quota-backend-bytes: 6442450944

listen-peer-urls: https://10.10.20.203:2380
listen-client-urls: https://10.10.20.203:2379,https://127.0.0.1:2379
initial-advertise-peer-urls: https://10.10.20.203:2380
advertise-client-urls: https://10.10.20.203:2379
initial-cluster: etcd01=https://10.10.20.201:2380,etcd02=https://10.10.20.202:2380,etcd03=https://10.10.20.203:2380

initial-cluster-token: 'etcd-cluster'
initial-cluster-state: 'new'

strict-reconfig-check: false
enable-pprof: true
proxy: 'off'

client-transport-security:
  cert-file: /k8s/etcd/ssl/server-cert.pem
  key-file: /k8s/etcd/ssl/server-key.pem
  trusted-ca-file: /k8s/etcd/ssl/ca.pem
  client-cert-auth: true

  auto-tls: false

peer-transport-security:
  cert-file: /k8s/etcd/ssl/peer-cert.pem
  key-file: /k8s/etcd/ssl/peer-key.pem
  trusted-ca-file: /k8s/etcd/ssl/ca.pem
  client-cert-auth: true

  auto-tls: false
  allowed-cn:
  allowed-hostname:

self-signed-cert-validity: 100
log-level: info
logger: zap
log-outputs: [stderr]
force-new-cluster: false
auto-compaction-mode: periodic
auto-compaction-retention: "1"

cipher-suites: [
  TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
]

tls-min-version: 'TLS1.2'
tls-max-version: 'TLS1.3'
EOF
```
## 手动生成SSL证书

> 参考另一篇文章《SSL证书生成步骤》，证书目录在 /k8s/etcd/ssl/

### 创建系统启动文件

```properties
sudo tee /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
# EnvironmentFile=/k8s/etcd/cfg/etcd
ExecStart=/k8s/etcd/bin/etcd --config-file=/k8s/etcd/cfg/etcd.conf
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

### 将启动文件和配置文件拷贝到 node02、node03

```properties
cd /k8s/etcd
scp -r bin node02:/k8s/etcd/
scp -r ssl node03:/k8s/etcd/
# 注意修改配置文件
scp /usr/lib/systemd/system/etcd.service node02:/usr/lib/systemd/system/etcd.service
scp /usr/lib/systemd/system/etcd.service node03:/usr/lib/systemd/system/etcd.service 
```

## 启动服务

- 注意：启动ETCD集群同时启动二个节点，启动一个节点集群是无法正常启动的

```properties
sudo systemctl daemon-reload && sudo systemctl enable etcd && sudo systemctl start etcd
sudo systemctl status etcd
sudo systemctl stop etcd
# 查看服务日志，看是否有错误信息，确保服务正常
journalctl -f -u etcd
# 更详细日志
tail -f /var/log/messages 
```



# 验证集群

```properties
etcdctl --cacert=/k8s/etcd/ssl/ca.pem --cert=/k8s/etcd/ssl/client-cert.pem --key=/k8s/etcd/ssl/client-key.pem --endpoints="https://10.10.20.201:2379,https://10.10.20.202:2379,https://10.10.20.203:2379"  -w table member list
--------------------
endpoint health

alarm list
lease list
role list
user list
auth status
endpoint status
```
> **注意：**
> 启动ETCD集群同时启动二个节点，启动一个节点集群是无法正常启动的

```properties
# 删除节点
member remove <ID>
# 增加节点
member add etcdXX --peer-urls=https://192.168.10.XXX:2380
```
### 查看日志

```properties
journalctl -f -u etcd
netstat -tunlp |grep 2379
ps -ef |grep etcd
```



# k8s数据迁移

### 【20】节点数据导出

#### 注意：先停止k0sController进程

```properties
systemctl stop k0scontroller
systemctl status k0scontroller

cd /k0s/etcd
# etcdctl --write-out=json get / > etcd_data.json
etcdctl --cacert=/var/lib/k0s/pki/etcd/ca.crt --cert=/var/lib/k0s/pki/apiserver-etcd-client.crt --key=/var/lib/k0s/pki/apiserver-etcd-client.key --endpoints="https://127.0.0.1:2379" snapshot save k8s_snap.db
# 查看快照
etcdutl snapshot status k8s_snap.db -w table
```

### 【40】节点数据导入

#### 注意：先停止ETCD进程

```properties
systemctl stop etcd
systemctl status etcd

cd /k0s/etcd
etcdutl snapshot restore k8s_snap.db --data-dir default
cd default

mv /data/etcd/data/member /data/etcd/data/member.bak
cp member /data/etcd/data/
```

#### 启动ETCD进程，验证集群同上



## Q&A:

1.  若没有成功启动服务，看到提示信息：member XXXXXX223XX has already been bootstrapped：请修改 `--initial-cluster-state=existing` 将new这个参数修改成existing，启动正常！