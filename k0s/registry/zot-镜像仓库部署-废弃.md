# zot-k8s 镜像仓库部署

官网：https://zotregistry.dev/v2.1.15/install-guides/install-guide-k8s/

GitHub: https://github.com/project-zot/zot

## 下载 zot helm chart

```properties
helm repo add project-zot http://zotregistry.dev/helm-charts
helm repo update project-zot
helm search repo project-zot -l
# ------下载最新版本-----------------------------------------------------
cd /k0s/zot
helm pull project-zot/zot --version 0.1.122
tar -zxf zot-0.1.122.tgz
```


## vim values.yaml

```properties
cd /k0s/zot/zot-0.1.122
# 修改镜像名
sudo vim /k0s/zot/zot-0.1.122/values.yaml
```

S3配置： https://zotregistry.dev/v2.1.18/articles/storage/#configuring-remote-storage-with-s3

集群配置：https://zotregistry.dev/v2.1.9/articles/scaleout/#cluster-member-configuration

```properties
replicaCount: 3
image:
  repository: 192.168.1.118:80/project-zot/zot
namespace: zot
service:
  type: ClusterIP
mountConfig: true
configFiles:
  config.json: |-
    {
      "storage": {
        "rootDirectory": "/var/lib/registry",
        "dedupe": false,
        "remoteCache": true,
        "storageDriver": {
            "name": "s3",
            "rootdirectory": "/zot-registry",
            "region": "us-east-1",
            "bucket": "gin-nid",
            "regionendpoint": "http://192.168.1.118:8333",
            "accesskey": "92433179-b9ae-758a-afe4-1db8a3ba7460",
            "secretkey": "6787877a-62f5-0455-e64a-c71f0f05d5cf",
            "forcepathstyle": true,
            "secure": false
        },
        "cacheDriver": {
          "name": "redis",
          "addr": ["192.168.1.99:6379"],
          "password": "zaq1xsw2",
          "db": 0,
          "pool_size": 10,
          "keyprefix": "zot"
        }
      },
      "http": { 
        "address": "0.0.0.0", 
        "port": "5000",
        "auth": {
          "htpasswd": {
            "path": "/secret/htpasswd"
          },
          "sessionDriver": {
            "name": "redis",
            "addr": ["192.168.1.99:6379"],
            "password": "zaq1xsw2",
            "db": 0,
            "pool_size": 10,
            "keyprefix": "zotsession"
          }
        } 
      },
      "log": { "level": "debug" },
      "extensions": {
        "search": {
          "enable": true
        },
        "ui": {
          "enable": true
        }
      }
    }
mountSecret: true
secretFiles:
  htpasswd: |-
    admin:$2y$05$vmiurPmJvHylk78HHFWuruFFVePlit9rZWGA/FbZfTEmNRneGJtha
    user:$2y$05$L86zqQDfH5y445dcMlwu6uHv.oXFgT6AiJCwpv3ehr7idc0rI3S2G
test:
  image:
    repository: 192.168.1.118:80/alpine
serviceHeadless:
  enabled: true
  port: 5000
persistence: false
pvc:
  create: true
  accessModes: ["ReadWriteOnce"]
  storage: 5Gi
  storageClassName: seaweedfs-storage
```

- storage.rootDirectory: **本地**目录，存放缓存数据库（如 bolt.db）和临时文件
- storage.storageDriver.rootdirectory: **S3 存储桶内**的路径前缀，所有镜像数据存于此路径下
- storageClassName: nfs-client

## 创建桶

```properties
aws --endpoint-url http://localhost:8333 s3 mb s3://zot-registry
```

## 创建 ns

```properties
k create ns zot
k label ns zot istio-injection=enabled --overwrite
```

##  本地安装

```properties
cd /k0s/zot
helm -n zot install zot ./zot-0.1.122 -f values.yaml --wait
```

## 检查校验

```properties
curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/v2/_zot/ext/mgmt"
curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/v2/_oci/ext/discover"
curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/v2/_catalog"
curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/v2/"
curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/zot/auth/apikey"

curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/v2/chrislusf/seaweedfs/manifests/sha256:fcdff3de097ca69f225523d76197b2500a0e278317bf6c88e75c1c38dcac19ff"
curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/v2/chrislusf/seaweedfs/manifests/latest"
curl -u admin:admin2026 "https://zot.develop.ekemp.com.cn/v2/chrislusf/seaweedfs/blobs/latest"
```





## ~~删除卸载~~

```properties
helm -n zot delete zot
# kubectl delete ns zot
# 删除桶
aws --endpoint-url http://localhost:8333 s3 rb s3://zot-registry
aws --endpoint-url http://localhost:8333 s3 mb s3://zot-registry
# 删除redis DB2
zot
zotsession
# 删除PVC
kubectl -n zot delete pvc zot-pvc-zot-0
kubectl -n zot delete pvc zot-pvc-zot-1
kubectl -n zot delete pvc zot-pvc-zot-2
```



## 开启签名

```json
"extensions": {
    "trust": {
        "enable": true,
        "cosign": true,
        "notation": true
    }
}
```

### Cosign 签名

- GitHub：https://github.com/sigstore/cosign/releases/download/v3.0.6/cosign-linux-amd64
- 最新版本：3.0.6， cosign version

```properties
vim ~/.bashrc
export COSIGN_PASSWORD=ekemp
source ~/.bashrc
# 生成秘钥对
cosign generate-key-pair
# 构建并推送镜像
docker push zot.develop.ekemp.com.cn/chrislusf/seaweedfs
docker push zot.develop.ekemp.com.cn/prom/prometheus:v2.21.0
# 获取 hash
docker inspect --format='{{index .RepoDigests 0}}' zot.develop.ekemp.com.cn/chrislusf/seaweedfs:latest
# 使用私钥签名镜像,并推送签名到 zot
cosign sign --key cosign.key zot.develop.ekemp.com.cn/chrislusf/seaweedfs@sha256:fcdff3de097ca69f225523d76197b2500a0e278317bf6c88e75c1c38dcac19ff
# 使用公钥验证签名
cosign verify --key cosign.pub zot.develop.ekemp.com.cn/chrislusf/seaweedfs@sha256:fcdff3de097ca69f225523d76197b2500a0e278317bf6c88e75c1c38dcac19ff


# 上传公钥到 Zot 自动验证
curl -v -X POST https://zot.develop.ekemp.com.cn/v2/_zot/ext/cosign \
-H "Content-Type: application/octet-stream" \
-u admin:admin2026 --data-binary @cosign.pub
# 查看文件
sudo ls -l /srv/nfs/mosip/dev/zot-zot-pvc-zot-2-pvc-3af73cb5-6b50-4e74-a011-7384486f87f1/_cosign/

curl -v -X POST http://10.43.141.206:5000/v2/_zot/ext/cosign \
-H "Content-Type: application/octet-stream" \
-u admin:admin2026 --data-binary @cosign.pub

curl -X POST -H "Content-Type: application/json" --data '{ "query": "{ ImageListForCVE (id:\"CVE-2002-1119\") { Results { RepoName Tag } } }" }' http://10.43.141.206:5000/v2/_zot/ext/search

```

### Notation 签名

```properties
# 生成证书
export NOTATION_EXPERIMENTAL=1
notation cert generate-test your-company.com --default
# 构建并推送镜像
# 获取镜像 digest 并签名
IMAGE_DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' your-registry.com/your-image:latest)
notation sign $IMAGE_DIGEST
# 验证签名
notation verify $IMAGE_DIGEST
```



