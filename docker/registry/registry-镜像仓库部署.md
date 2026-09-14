# registry-k8s 镜像仓库部署

docker hub：https://hub.docker.com/registry

GitHub: https://github.com/distribution/distribution

## 增加基础认证用户

```properties
# -B 参数指定使用 bcrypt 加密，-c 表示创建新文件: admin2026
htpasswd -Bc auth/htpasswd admin
# 添加更多用户（比如 developer），不要再使用 -c 参数: ekemp2026
htpasswd -B auth/htpasswd ekemp
# 创建 Kubernetes Secret
kubectl -n registry create secret generic registry-htpasswd --from-file=auth/htpasswd
```

## 调整nginx.conf

```properties
http {
    # 对于 Docker Registry，应该设置足够大
    client_max_body_size 0;  # 0 表示不限制（推荐）
    # 或者设置一个很大的值
    # client_max_body_size 1024m;  # 1GB
}
```

## 创建桶

```properties
aws --endpoint-url http://localhost:8333 s3 mb s3://docker-registry
```

## 创建 ns

```properties
k create ns registry
k label ns registry istio-injection=enabled --overwrite
```

## vim registry-config.yaml

```properties
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-config
  namespace: registry
data:
  config.yml: |
    version: 0.1
    log:
      level: info
      fields:
        service: registry
    storage:
      delete:
        enabled: true
      s3:
        region: us-east-2
        bucket: docker-registry
        regionendpoint: http://192.168.1.118:8333
        secure: false
        encrypt: false
        chunksize: 5242880
        forcepathstyle: true
    auth:
      htpasswd:
        realm: basic-realm
        path: /auth/htpasswd
    http:
      addr: :5000
      secret: 8fbe7b0feb7caa0b7f48594591b21c3d
      draintimeout: 60s
      headers:
        X-Content-Type-Options: [nosniff]
        Access-Control-Allow-Origin: ['*']
        Access-Control-Allow-Headers: ['Accept', 'Cache-Control']
        Access-Control-Allow-Methods: ['HEAD', 'GET', 'OPTIONS', 'DELETE']
        Access-Control-Expose-Headers: ['Docker-Content-Digest']
    health:
      storagedriver:
        enabled: true
        interval: 10s
        threshold: 3
```

## s3-secret

```properties
apiVersion: v1
kind: Secret
metadata:
  name: registry-s3-secret
  namespace: registry
type: Opaque
stringData:
  accesskey: 92433179-b9ae-758a-afe4-1db8a3ba7460
  secretkey: 6787877a-62f5-0455-e64a-c71f0f05d5cf
```

## service

```properties
apiVersion: v1
kind: Service
metadata:
  name: docker-registry
  namespace: registry
spec:
  selector:
    app: docker-registry
  ports:
  - port: 5000
    targetPort: 5000
    name: registry
  type: ClusterIP
  clusterIP: 10.43.100.50
```

## deploy

```properties
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry
  namespace: registry
  labels:
    app: docker-registry
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: docker-registry
  template:
    metadata:
      labels:
        app: docker-registry
    spec:
      containers:
      - name: registry
        image: 192.168.1.118:80/registry:3.1.0
        ports:
        - containerPort: 5000
          name: registry
        env:
        - name: REGISTRY_STORAGE_S3_ACCESSKEY
          valueFrom:
            secretKeyRef:
              name: registry-s3-secret
              key: accesskey
        - name: REGISTRY_STORAGE_S3_SECRETKEY
          valueFrom:
            secretKeyRef:
              name: registry-s3-secret
              key: secretkey
        volumeMounts:
        - name: registry-config
          mountPath: /etc/distribution/config.yml
          subPath: config.yml
          readOnly: true
        - name: registry-htpasswd
          mountPath: /auth
          readOnly: true
        resources:
          requests:
            memory: "1024Mi"
            cpu: "1"
          limits:
            memory: "2056Mi"
            cpu: "2"
       volumes:
      - name: registry-config
        configMap:
          name: registry-config
      - name: registry-htpasswd
        secret:
          secretName: registry-htpasswd
```


## 应用部署

### 创建用于仓库认证的 Secret

```properties
kubectl -n ekemp create secret docker-registry ekemp-registry-secret \
  --docker-server=10.43.100.50:5000 \
  --docker-username=admin \
  --docker-password=admin2026
```

### 将 Secret 添加到默认服务账户

- `default` 

```properties
kubectl -n ekemp edit serviceaccount ekemp-sa
#-------------------------------------------
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ekemp-sa
  namespace: ekemp
imagePullSecrets:
- name: ekemp-registry-secret 
# 这里填写你第一步创建的 Secret 名字
```

## 垃圾回收

```properties
/bin/registry garbage-collect /etc/distribution/config.yml -m
```

## 删除卸载~~

```properties
kubectl delete ns registry
```








