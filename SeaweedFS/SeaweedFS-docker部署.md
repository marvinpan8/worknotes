# docker-compose部署 SeaweedFS

- **版本：4.19**
- wiki: https://github.com/seaweedfs/seaweedfs/wiki

### 网络与端口

| 服务           | HTTP端口                     | gRPC端口 |
| -------------- | ---------------------------- | -------- |
| **Master**     | **9333**                     | 19333    |
| **Volume**     | **8080**                     | 18080    |
| **Filer**      | **8888**                     | 18888    |
| **S3**         | **8333/8181(iceberg)**       | 18333    |
| **webdav**     | **7333**                     |          |
| **prometheus** | **9324/9325/9326/9327/9090** |          |

## docker compose部署

https://raw.githubusercontent.com/chrislusf/seaweedfs/master/docker/seaweedfs-compose.yml

#### cd ~/weed

```properties
services:
  master:
    image: chrislusf/seaweedfs # use a remote image
    ports:
      - 9333:9333
      - 19333:19333
      - 9324:9324
    command: 'master -ip=192.168.1.118 -ip.bind=0.0.0.0 -metricsPort=9324'
    volumes:
      - /mnt/dockerImageData/weed:/data
  volume:
    image: chrislusf/seaweedfs # use a remote image
    ports:
      - 8080:8080
      - 18080:18080
      - 9325:9325
    command: 'volume -ip=192.168.1.118 -master="master:9333" -ip.bind=0.0.0.0 -port=8080 -metricsPort=9325'
    volumes:
      - /mnt/dockerImageData/weed:/data
    depends_on:
      - master
  filer:
    image: chrislusf/seaweedfs # use a remote image
    ports:
      - 8888:8888
      - 18888:18888
      - 9326:9326
    command: 'filer -ip=192.168.1.118 -master="master:9333" -ip.bind=0.0.0.0 -metricsPort=9326'
    volumes:
      - /mnt/dockerImageData/weed:/data
    tty: true
    stdin_open: true
    depends_on:
      - master
      - volume
  s3:
    image: chrislusf/seaweedfs # use a remote image
    ports:
      - 8333:8333
      - 8181:8181
      - 9327:9327
    command: 's3 -filer="192.168.1.118:8888" -ip.bind=0.0.0.0 -metricsPort=9327 -config=/etc/seaweedfs/s3.json'
    volumes:
      - /mnt/dockerImageData/weed:/data
      - ./s3.json:/etc/seaweedfs/s3.json
    depends_on:
      - master
      - volume
      - filer
  webdav:
    image: chrislusf/seaweedfs # use a remote image
    ports:
      - 7333:7333
    command: 'webdav -filer="192.168.1.118:8888"'
    volumes:
      - /mnt/dockerImageData/weed:/data
    depends_on:
      - master
      - volume
      - filer
  prometheus:
    image: prom/prometheus:v2.21.0
    ports:
      - 9090:9090
    volumes:
      - /mnt/dockerImageData/prometheus:/prometheus
      - ./prometheus:/etc/prometheus
    command: '--web.enable-lifecycle --config.file=/etc/prometheus/prometheus.yml'
    depends_on:
      - s3
```

- **ip 必须写主机IP**

### vim prometheus/prometheus.yml

```properties
global:
  scrape_interval: 30s
  scrape_timeout: 10s

scrape_configs:
  - job_name: services
    metrics_path: /metrics
    static_configs:
      - targets:
          - 'prometheus:9090'
          - 'master:9324'
          - 'volume:9325'
          - 'filer:9326'
          - 's3:9327'
```

### 启动

```properties
# 因promethes 是 nobody 用户启动
sudo chown -R nobody:nogroup /mnt/dockerImageData/prometheus
# 启动
docker compose up -d
```

### 验证检查
```properties
# 验证集群
curl http://192.168.1.118:9333/cluster/status?pretty=y
# 查看集群容量和 Volume 分布情况
curl http://192.168.1.118:9333/dir/status?pretty=y
# 查看所有 Volume 的详细信息（ID、大小、副本策略等）
curl http://192.168.1.118:9333/vol/status?pretty=y
# 检查 单个 Volume 状态
curl http://192.168.1.118:8080/status?pretty=y

# 测试上传 副本策略 000
curl "http://localhost:9333/dir/assign?replication=000"
# 返回 {"fid":"3,016e37c6e6","url":"192.168.1.101:8080"}
curl -X PUT -F "file=@test.txt" "http://192.168.1.101:8080/3,016e37c6e6"
# 监控磁盘 IO
iostat -x 5
# 观察 %util 和 await
# 如果 %util < 70%，可适当调高,ompactionMBps=80

# 进入容器
docker exec -it weed-s3-1 sh
# weed shell 查看
weed shell -filer="filer:8888" -master="master:9333"
> volume.list
# 手动垃圾回收
> volume.vacuum
# 每周执行数据完整性检查
> volume.fsck -collection=all
# 创建 匿名用户，指定 s3桶
> s3.bucket.access -name test-bucket -user anonymous -access Read,List
# 查看匿名用户权限
> s3.bucket.access -name test-bucket -user anonymous
# 删除 匿名用户，指定 s3桶
> s3.bucket.access -name test-bucket -user anonymous -access none
```

- **volumeSizeLimit**: 每个卷的**大小上限**（这里是 1024 MB）。这是查看总容量时的核心参考值。
- **volumes:14/340**: 当前已分配了 14 个卷，最多可分配 340 个。

- **最大容量：340 (个) × 1 (GB/个) = 340 GB**，`weed shell` 可以动态增加卷的数量，无需重启服务。

---

## 测试S3

```properties
sudo snap install aws-cli --classic
aws --version
# 配置 AWS CLI 使用测试凭证
export AWS_ACCESS_KEY_ID=92433179-b9ae-758a-afe4-1db8a3ba7460
export AWS_SECRET_ACCESS_KEY=6787877a-62f5-0455-e64a-c71f0f05d5cf
# 测试访问（不再需要 --no-sign-request）
aws --endpoint-url http://localhost:8333 s3 ls

# 创建桶
aws --endpoint-url http://localhost:8333 s3 mb s3://test-bucket
# 上传文件
echo "test" > test.txt
aws --endpoint-url http://localhost:8333 s3 cp test.txt s3://test-bucket/
# 列出文件
aws --endpoint-url http://localhost:8333 s3 ls s3://test-bucket/
# 查看 s3 元数据
aws --endpoint-url http://localhost:8333 s3api head-object --bucket test-bucket --key test.txt

# 清理文件，查看 mysql，删除记录了
aws --endpoint-url http://localhost:8333 s3 rm s3://test-bucket/test.txt
# 清理文件，查看 mysql，删除表了
aws --endpoint-url http://localhost:8333 s3 rb s3://test-bucket
```

###  查看实际存储磁盘

Fid 由三个部分组成 【VolumeId, NeedleId, Cookie】

- VolumeId: 14 32bit 存储的物理卷的Id
- NeedleId: f4 64bit 全局唯一NeedleId，每个存储的文件都不一样(除了互为备份的)。
- Cookie: 187c8fdb 32bit Cookie值，为了安全起见，防止恶意攻击。

```properties
# 1. 查询 filer 元数据， 查看 fid.volume_id = 9; file_id = 9,02227989d5
curl -H "Accept: application/json" "http://localhost:8888/buckets/test-bucket/test.txt?metadata=true&pretty=y"
# 2. 向 Master 查询 volumeId = 9 所在Volume 即 物理磁盘位置 102 102 104
curl "http://localhost:9333/dir/lookup?volumeId=9&pretty=y"
# 3. 根据以上所在 Volume 查看Collection 和 FileCount 
curl "http://localhost:8080/status?pretty=y"
{
      "Id": 9,
      "Size": 27,
      "ReplicaPlacement": {},
      "Ttl": {
        "Count": 0,
        "Unit": 0
      },
      "DiskType": "",
      "DiskId": 0,
      "Collection": "test-bucket",
      "Version": 3,
      "FileCount": 1,
      "DeleteCount": 0,
      "DeletedByteCount": 0,
      "ReadOnly": false,
      "CompactRevision": 0,
      "ModifiedAtSecond": 0,
      "RemoteStorageName": "",
      "RemoteStorageKey": ""

}   
# 3. 直接通过 filer API 访问文件
curl -H "Accept: application/json" http://localhost:8080/9,02227989d5

# 都在  test-bucket_7.dat 中
sudo ls -l /mnt/dockerImageData/weed/
```

## 删除卸载

1. **停止所有服务**：停止所有机器上的 Master、Volume Server、Filer 服务。
```properties
cd ~/weed
docker compose down
```

2. **清理数据目录**：

```properties
# Master：删除 -mdir 参数指定的目录下的所有内容。
sudo rm -rf /mnt/dockerImageData/weed/*
# Volume Server：删除 -dir 参数指定的所有目录下的内容。
sudo rm -rf /mnt/dockerImageData/prometheus/*
```



## k8s-csi 集成

获取最新版  deploy/kubernetes/seaweedfs-csi.yaml：https://github.com/seaweedfs/seaweedfs-csi-driver.git

```properties
# 替换 SEAWEEDFS_FILER 地址（两处）
SEAWEEDFS_FILER:8888  -> 10.10.10.104:8888
# 替换 namespace
namespace: default  ->  namespace: kube-system
namespace=default  ->  namespace=kube-system
# 部署执行
kubectl apply -f deploy/kubernetes/seaweedfs-csi.yaml
# 检查
kubectl get po -n kube-system
```

### 测试

```properties
kubectl apply -f deploy/kubernetes/sample-seaweedfs-pvc.yaml
# 查看 volumeName: pvc-d620b253-dd28-46cd-b8ad-fb765a891b2a
kubectl get pvc
kubectl apply -f deploy/kubernetes/sample-busybox-pod.yaml
kubectl exec my-csi-app -- df -h
#-------------------------------------------------------
10.10.10.104:8888:/buckets/pvc-d620b253-dd28-46cd-b8ad-fb765a891b2a
                          1.1P         0      1.1P   0% /data
```