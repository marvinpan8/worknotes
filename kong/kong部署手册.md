## 安装PG数据库

- 创建PVC 
- 执行postgres.yaml，创建RC-PG，创建SVC-PG

## 创建secret

执行此文件 setup_certificate.sh

```bash
#!/bin/bash

set -eufo pipefail

mkdir tmp-qweiur3412
cd tmp-qweiur3412

### Create a key+certificate for the control plane
cat <<EOF | kubectl create -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: kong-control-plane.kong.svc
spec:
  request: $(openssl req -new -nodes -batch -keyout privkey.pem -subj /CN=kong-control-plane.kong.svc | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
kubectl certificate approve kong-control-plane.kong.svc
kubectl -n kong create secret tls kong-control-plane.kong.svc --key=privkey.pem --cert=<(kubectl get csr kong-control-plane.kong.svc -o jsonpath='{.status.certificate}' | base64 --decode)
kubectl delete csr kong-control-plane.kong.svc
rm privkey.pem
cd ..
rm -rf tmp-qweiur3412
```

## 创建用户角色权限

执行 kong-sa-role.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: kong
  name: kong
  labels:
    app: kong
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  namespace: kong
  name: kong
  labels:
    app: kong
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  namespace: kong
  name: kong
  labels:
    app: kong
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kong
subjects:
- kind: ServiceAccount
  namespace: kong
  name: kong
```

## 创建控制平面

- 执行 control-plane-deploy.yaml
- 执行 control-plane-svc.yaml

## 创建数据平面

- 执行 data-plane-deploy.yaml
- 执行 data-plane-service.yaml

## 创建Konga控制台

- konga-ui.yml 修改环境变量 NODE_ENV=production，如果想开发环境则值为development
- 执行 konga-ui.yml
- 内网访问或者配置ingress访问
- 密码采用的是 java 的 BCryptPasswordEncoder加密器

## 创建service,route,consumer并绑定hmac
## 用户绑定服务
```bash
curl http://172.22.254.158:8001/consumer-services
curl http://172.22.254.158:8001/consumers
curl http://172.22.254.158:8001/services
curl http://172.22.254.158:8001/consumer-services -H 'Content-Type: application/json' -X POST -d '{"created_at":1579659182,"consumer":{"id":"58c310c9-9fcc-4d81-874b-a477b71d18e0"},"service":{"id":"f3f389fb-f6d0-44a4-bba0-bc5afb5e3d4e"},"name":"jrtzretail$sdp-rest","expired_at":1737511982}'
```
