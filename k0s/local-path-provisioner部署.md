# local-path-provisioner 部署

- GitHub:  https://github.com/rancher/local-path-provisioner.git

### values.yaml

```properties
image:
  repository: 10.10.10.102:5000/rancher/local-path-provisioner
  tag: v0.0.36
helperImage:
  repository: 10.10.10.102:5000/library/busybox
  tag: 1.38.0
# 使用 storageClassConfigs 定义两个存储类
storageClass:
  provisionerName: rancher.io/local-path-provisioner
storageClassConfigs:
  # 第一个存储类：回收策略为 Delete (自动清理)
  local-path-delete:
    storageClass:
      create: true
      name: local-path-delete
      defaultClass: false
      reclaimPolicy: Delete
      volumeBindingMode: WaitForFirstConsumer
      defaultVolumeType: local
    nodePathMap:
      - node: DEFAULT_PATH_FOR_NON_LISTED_NODES
        paths:
          - /data/pvs/delete
          
  # 第二个存储类：回收策略为 Retain (保留数据)
  local-path-retain:
    storageClass:
      create: true
      name: local-path-retain
      defaultClass: false
      reclaimPolicy: Retain
      volumeBindingMode: WaitForFirstConsumer
      defaultVolumeType: local
    nodePathMap:
      - node: DEFAULT_PATH_FOR_NON_LISTED_NODES
        paths:
          - /data/pvs/retain
helperPod:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

## helm 部署

```properties
git clone https://github.com/rancher/local-path-provisioner.git
cd local-path-provisioner
helm -n seaweedfs install local-path-storage ./local-path-provisioner -f values-local-path.yaml --wait
# 删除
# helm -n seaweedfs delete local-path-storage
```

### 验证

```properties
kubectl get StorageClass
```

