# zot-镜像仓库二进制部署

**版本：v2.1.21**

**官网**：https://zotregistry.dev/v2.1.21/install-guides/install-guide-k8s/  

**GitHub:** https://github.com/project-zot/zot

## ※ 下载 zot

**二进制：**https://github.com/project-zot/zot/releases/download/v2.1.21/zot-linux-amd64

```properties
sudo mkdir -p /data/zot/{data,logs,sync}

sudo mv zot-linux-amd64 /usr/local/bin/zot
sudo chmod +x /usr/local/bin/zot
sudo chown root:root /usr/local/bin/zot
# zli
sudo mv zli-linux-amd64 /usr/local/bin/zli
sudo chmod +x /usr/local/bin/zli
sudo chown root:root /usr/local/bin/zli
# 查看
ls -l /usr/local/bin
zot -v
zli -v
```

## ※ 创建配置 config.json

**S3配置：** https://zotregistry.dev/v2.1.21/articles/storage/#configuring-remote-storage-with-s3

**集群配置：**https://zotregistry.dev/v2.1.9/articles/scaleout/#cluster-member-configuration

**集群样例** https://zotregistry.dev/v2.1.18/articles/scaleout/#using-zot-web-ui-and-zli-in-a-scale-out-cluster

#### sudo vim /etc/zot/config.json

```properties
sudo cp config.json /etc/zot/
sudo cp htpasswd /etc/zot/
sudo chown -R root:root /etc/zot
# 校验
sudo -u zot zot verify /etc/zot/config.json
```


- **commit: false, 是否不缓存立即刷盘**

- **dedupe:  false ⚠️关闭镜像层layer去重，无状态集群服务应关闭⚠️**

- **gc: true  启动垃圾回收**

- **gcDelay: 1h  首次启动后多久执行垃圾回收**

- **gcInterval: 1h  间隔多久执行垃圾回收**

- **gcTimeWindow: "01:00-08:00" 垃圾回收事件窗口**

- **fastRestart: false  默认重启扫描**

- **redirectBlobURL: false  全局默认关闭S3重定向，签名并缓存镜像数据**

- **remoteCache: true ⚠️使用远程缓存元数据**

- **cacheDriver: 当 dedupe=true 开启去重镜像层时，专门存储和管理重复 Blob 的元数据索引，而不是存储 Blob 数据本身**

- **storageDriver**

  - **rootDirectory S3前缀, ⚠️千万不要设置，否则zli repo list 获取不到：系S3双重路径所致BUG**
  - **loglevel: info  S3客户端日志级别**

- **cluster:  无状态服务**

- **http**

  - **compat: ["docker2s2"]**   **docker兼容模式**

- **extensions**

  - **sync**   **集群模式分片hash同步**

  - **scrub 存储清洗，检查所有数据块（blobs）是否完好、可读**
  - **search**  
    - **cve: 远程S3服务不支持开启漏洞扫描**
    - **漏洞库默认从ghcr.io/aquasecurity/trivy-db 和 ghcr.io/aquasecurity/trivy-java-db 下载**
  - **search.cve.trivy.ignoreFile** **忽略的漏洞列表**

## ※ 创建用户密码

```properties
sudo apt install apache2-utils -y
# 创建第一个用户
sudo sh -c "htpasswd -bnB admin admin123 > /etc/zot/htpasswd"
# 新增其他用户
sudo sh -c "htpasswd -bnB ekemp ekemp123 >> /etc/zot/htpasswd"
# 创建系统用户，不创建加目录，进制shell密码登录
sudo adduser --no-create-home --disabled-password --gecos --disabled-login zot
# 授权目录
sudo chown -R zot:zot /data/zot
sudo chown -R root:root /etc/zot/
sudo ls -l /etc/zot/
```

## ※ 系统启动文件

```properties
sudo tee /etc/systemd/system/zot.service << EOF
[Unit]
Description=OCI Distribution Registry
Documentation=https://zotregistry.dev/
After=network.target auditd.service local-fs.target

[Service]
Type=simple
ExecStart=/usr/local/bin/zot serve /etc/zot/config.json
Restart=on-failure
User=zot
Group=zot
LimitNOFILE=500000
MemoryHigh=30G
MemoryMax=32G

[Install]
WantedBy=multi-user.target
EOF
```
## 启动
```properties
ls -l /etc/systemd/system/zot.service
sudo systemctl daemon-reload
sudo systemctl enable zot
sudo systemctl is-enabled zot
# 启动
sudo systemctl start zot
sudo systemctl restart zot
# 查看停止
sudo systemctl status zot
sudo systemctl stop zot
# 日志
sudo netstat -tunlp |grep 5000
sudo tail -f -n 500 /data/zot/logs/zot.log
# 拷贝日志
sudo cp /data/zot/logs/zot.log .
sudo chown ekemp:ekemp zot.log
```

## 删除清理

- **停止服务，先删除 seaweedfs 的桶 zot-data**
- **再删除redis 以{zot}开头的数据，指定同一个hash槽**
- **最后删除本地目录**

```properties
sudo ls -l /data/zot/data
sudo ls -l /data/zot/logs
sudo ls -l /data/zot/sync
sudo rm -rf /data/zot/data/*
sudo rm -rf /data/zot/logs/*
sudo rm -rf /data/zot/sync/*
```

## zli工具
```properties
zli config add local http://10.10.20.201:5000
zli config --list
zli config set-default local
zli config clear-default
#---------------------------------------------
zli repo list --config local -u admin:admin123
zli image list --config local -u admin:admin123
# curl
curl -u admin:admin123 http://10.10.20.201:5000/v2/_catalog
curl -u admin:admin123 http://10.10.20.201:5000/v2/_oci/ext/discover
curl -u admin:admin123 http://10.10.20.201:5000/v2/ekemp/ekemp-nid/tags/list
```

## k8s配置

```properties
# 已经登录过节点,另外一种方式创建
kubectl create secret generic zot-ekemp --from-file=.dockerconfigjson=/home/ekemp/.docker/config.json --type=kubernetes.io/dockerconfigjson
# 修改指定SA
kubectl patch serviceaccount gin-prod-sa -p '{"imagePullSecrets": [{"name": "zot-ekemp"}]}' --namespace=gin-prod
# 修改默认SA
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "zot-ekemp"}]}' --namespace=gin-prod
```






## ※ 配置样例

```properties
{
  "distSpecVersion": "1.1.1",
  "storage": {
    "rootDirectory": "/data/zot/data",
    "commit": true,
    "gc": true,
    "gcDelay": "1h",
    "gcInterval": "2h",
    "gcTimeWindow": "22:00-03:00",
    "dedupe": true,
    "remoteCache": true,
    "cacheDriver": {
      "name": "redis",
      "addr": [
        "10.10.20.201:6379",
        "10.10.20.202:6379",
        "10.10.20.203:6379",
        "10.10.10.101:6379",
        "10.10.10.103:6379",
        "10.10.10.105:6379"
      ],
      "password": "zaq1xsw2",
      "pool_size": 10,
      "keyprefix": "zot/dedupe",
      "dial_timeout": "5s",
      "read_timeout": "5s",
      "write_timeout": "5s"
    },
     "storageDriver": {
      "name": "s3",
      "rootdirectory": "/zot",
      "region": "us-east-1",
      "bucket": "zot-data",
      "regionendpoint": "http://10.10.10.66:9333",
      "accesskey": "CM3T9PBVDBPS6NRKRU5B",
      "secretkey": "hawSs0K2WcvuRGmiu0z1R06SwiSmBoSD1P0mHPnHsQ",
      "forcepathstyle": true,
      "secure": false,
      "skipverify": true,
      "loglevel": "info"
    }
  },
  "cluster": {
    "members": [
      "10.10.20.201:5000",
      "10.10.20.202:5000",
      "10.10.20.203:5000"
    ],
    "hashKey": "loremipsumdolors"
  },
  "http": {
    "address": "0.0.0.0",
    "port": "5000",
    "realm":"zot",
    "compat": ["docker2s2"],
    "auth": {
      "htpasswd": {
        "path": "/secret/htpasswd"
      },
       "sessionDriver": {
        "name": "redis",
        "addr": [
          "10.10.20.201:6379",
          "10.10.20.202:6379",
          "10.10.20.203:6379",
          "10.10.10.101:6379",
          "10.10.10.103:6379",
          "10.10.10.105:6379"
        ],
        "password": "zaq1xsw2",
        "pool_size": 10,
        "keyprefix": "zot/session",
        "dial_timeout": "5s",
        "read_timeout": "5s",
        "write_timeout": "5s"
      }
    }
  },
  "log": {
    "level": "info",
    "audit": "/data/zot/logs/zot-audit.log",
    "output": "/data/zot/logs/zot.log"
  },
  "extensions": {
    "search": {
      "enable": true,
      "cve": {
        "updateInterval": "24h",
        "trivy": {
          "dbRepository": "ghcr.io/aquasecurity/trivy-db",
          "javaDBRepository": "ghcr.io/aquasecurity/trivy-java-db",
          "vulnSeveritySources": ["auto"],
          "sbom": {
            "enable": true,
            "format": "spdx-json"
          }
        }
      }
    },
     "ui": {
      "enable": true
    },
    "scrub": {
      "enable": true,
      "interval": "24h"
    },
    "metrics": {
      "enable": true,
      "prometheus": {
          "path": "/metrics"
      }
    }
  }
}
```

