# drone-k8s安装

### 创建namespace

```bash
kubectl create ns drone
```

### 创建PVC卷

```bash
cat << EOF | tee drone-pvc.yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: drone-server-sqlite-db
  namespace: drone
  annotations:
    volume.beta.kubernetes.io/storage-class: "glusterfs"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 25Gi
EOF
```

### 创建drone-secret.yaml

- 创建原始drone-secret.yaml

```bash
cat << EOF | tee drone-secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: drone-secrets
  namespace: drone
data:
  server.secret: REPLACE-THIS-WITH-BASE64-ENCODED-VALUE
EOF
```

- 生成token,base64编码，最后创建secret

```bash
# 生成 drone_token
drone_token=`cat /dev/urandom | env LC_CTYPE=C tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1`
b64_drone_token=`echo $drone_token | base64`
sed -e "s/REPLACE-THIS-WITH-BASE64-ENCODED-VALUE/${b64_drone_token}/g" -i drone-secret.yml
kubectl create -f drone-secret.yml
```

### 创建drone-configmap.yaml

