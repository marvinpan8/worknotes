## seaweedfs-csi-driver 集成 k8s（仅测试）

**仅测试原因：因 seaweedfs 使用的是 HDD机械硬盘，性能不合适**

```properties
git clone https://github.com/seaweedfs/seaweedfs-csi-driver.git
git fetch origin tag v1.4.26
git checkout -b branch_v1.4.26 v1.4.26
```

# ■■■ 1. helm 部署

- **101 节点**

### values.yaml

- **调用的是 filter grpc API**

```properties
seaweedfsFiler: "10.10.10.103:8888,10.10.10.104:8888,10.10.10.105:8888"
isDefaultStorageClass: true

csiProvisioner:
  image: 10.10.10.102:5000/sig-storage/csi-provisioner:v3.5.0
csiResizer:
  image: 10.10.10.102:5000/sig-storage/csi-resizer:v1.8.0
csiAttacher:
  enabled: false
csiNodeDriverRegistrar:
  image: 10.10.10.102:5000/sig-storage/csi-node-driver-registrar:v2.8.0
csiLivenessProbe:
  image: 10.10.10.102:5000/sig-storage/livenessprobe:v2.10.0
seaweedfsCsiPlugin:
  image: 10.10.10.102:5000/chrislusf/seaweedfs-csi-driver
  # tag defaults to Chart.appVersion when not set (v1.4.26)
mountService:
  image: 10.10.10.102:5000/chrislusf/seaweedfs-mount
  # tag defaults to Chart.appVersion when not set (v1.4.26)

node:
  # 驱动Pod可看到宿主机所有进程，自动修复所有业务Pod的FUSE挂载
  hostPID: true
  # 驱动 Pod 不会被自动重启。只有手动执行kubectl delete pod 删除某个节点上的驱动 Pod 时，它才会重建。
  # 这样你可以逐个节点、在业务低峰期、确认迁移完成后再操作，实现零停机升级。
  updateStrategy:
    type: OnDelete
  ## Change if not using standard kubernetes deployments, like k0s
  volumes:
    registration_dir: /data/kubelet/plugins_registry
    plugins_dir: /data/kubelet/plugins
    pods_mount_dir: /data/kubelet/pods
```

### 部署【101】

```properties
kubectl create ns seaweedfs
cd /k0s/weed
helm -n seaweedfs install seaweedfs-csi-driver ./seaweedfs-csi-driver -f values-seaweedfs-csi.yaml
# 删除
# helm -n seaweedfs delete seaweedfs-csi-driver
```
# ■■ 2. 简化版部署（跳过）

- 获取最新版（v1.4.26） deploy/kubernetes/seaweedfs-csi.yaml：

```properties
# 从helm chart 模板 产生 seaweedfs-csi.yaml
$ helm template seaweedfs ./deploy/helm/seaweedfs-csi-driver > deploy/kubernetes/seaweedfs-csi.yaml
# 替换 SEAWEEDFS_FILER 地址（两处）
SEAWEEDFS_FILER:8888  -> 10.10.10.104:8888
# 替换 namespace
namespace: default  ->  namespace: kube-system
namespace=default  ->  namespace=kube-system
# 查看 kubelet 的根目录
ps -ef | grep kubelet | grep root-dir
#  kubelet 的根目录 /var/lib/kubelet 替换为 /data/kubelet
sed 's+/var/lib/kubelet+/data/kubelet+g'  deploy/kubernetes/seaweedfs-csi.yaml
# 部署执行
kubectl apply -f deploy/kubernetes/seaweedfs-csi.yaml
# 检查
kubectl -n kube-system get po
```

### 测试

```properties
k get StorageClass
k apply -f deploy/kubernetes/sample-seaweedfs-pvc.yaml
# 查看 volumeName: pvc-d620b253-dd28-46cd-b8ad-fb765a891b2a
k get pvc
k apply -f deploy/kubernetes/sample-busybox-pod.yaml
k -n kube-system exec my-csi-app -- df -h
#-------------------------------------------------------
10.10.10.104:8888:/buckets/pvc-d620b253-dd28-46cd-b8ad-fb765a891b2a
                          1.1P         0      1.1P   0% /data
# 在宿主机上查看所有连接
ss -tn state established '( dport = :8888 or dport = :18888 )'
```

### mount pod

```properties
weed -logtostderr=true mount -dirAutoCreate=true -umask=000 -dir=/data/kubelet/plugins/kubernetes.io/csi/seaweedfs-csi-driver/88e44e2855e7dbd102ed80bb9aefe95c28cf3a40e301fc6fc02c2757b16488c4/globalmount -localSocket=/var/lib/seaweedfs-mount/seaweedfs-mount-88e44e2855e7dbd1.sock -cacheDir=/var/cache/seaweedfs/88e44e2855e7dbd102ed80bb9aefe95c28cf3a40e301fc6fc02c2757b16488c4 -concurrentWriters=128 -filer.path=/buckets/pvc-19e222bd-a871-401f-ab08-789f502f9528 -cacheCapacityMB=0 -collectionQuotaMB=51200 -filer=10.10.10.103:8888,10.10.10.104:8888,10.10.10.105:8888 -concurrentReaders=128 -collection=pvc-19e222bd-a871-401f-ab08-789f502f9528 -cacheMetaTtlSec=60
```



### 删除

```properties
kubectl delete -f deploy/kubernetes/sample-busybox-pod.yaml
kubectl delete -f deploy/kubernetes/sample-seaweedfs-pvc.yaml
kubectl delete -f deploy/kubernetes/seaweedfs-csi.yaml
```

### ■■ 3. StorageClass

- **默认StorageClass： seaweedfs-storage：**驱动程序会为每个请求创建单独的文件夹（`/buckets/<volume-id>`）并使用单独的 collection（`volume-id`）

- **指定集合 StorageClass：**有时需要确切的**集合名称**或更改**复制选项**。可以通过创建带有相应选项的单独 StorageClass 来实现：

```properties
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: seaweedfs-special
provisioner: seaweedfs-csi-driver
parameters:
  collection: mycollection
  replication: "011"
  diskType: "hdd"
```

### ■■ 4. 多pod 共享配置

当需要从不同的 Pod 访问**同一个文件夹**，并且需要拥有只读/读写权限。此时**使用默认的 StorageClass：seaweedfs-storage，且只需要创建 PV即可（不需要 namespace）**：

```properties
apiVersion: v1
kind: PersistentVolume
metadata:
  name: seaweedfs-static
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 100Mi
  csi:
    driver: seaweedfs-csi-driver
    # volumeHandle: /buckets/pvc-d620b253-dd28-46cd-b8ad-fb765a891b2a
    volumeHandle: seaweedfs-static-dfs-test
    volumeAttributes:
      collection: seaweedfs-static-test
      replication: "011"
      path: /test/temp
      diskType: "hdd"
    readOnly: true
  persistentVolumeReclaimPolicy: Retain
  storageClassName: seaweedfs-storage
  volumeMode: Filesystem
```

**将PV 绑定到指定 namespace 的 PVC：**

```properties
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: seaweedfs-static
  namespace: ekemp
spec:
  storageClassName: seaweedfs-storage
  volumeMode: Filesystem
  volumeName: seaweedfs-static
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

# local StorageClass

```properties
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

```properties
apiVersion: v1
kind: PersistentVolume
metadata:
  labels:
    loki-write-pv: "true" 
  name: loki-write-pv-0
spec:
  capacity:
    storage: 10Gi                      
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage     
  local:
    path: /data/loki-write-pv-0              
  nodeAffinity:                        
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - app101              
```