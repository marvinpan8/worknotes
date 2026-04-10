----
# 一、k8s 一键部署apisix gateway 和 ingress-controller 

官网helm：https://apache.github.io/apisix-helm-chart/docs/en/latest/apisix.html

https://docs.api7.ai/ingress-controller/set-up-ingress-controller-and-gateway/#install-apisix-and-apisix-ingress-controller

apisix:  https://apisix.apache.org/docs/apisix/deployment-modes/

ingress-controller: https://apisix-website-static.apiseven.com/docs/ingress-controller/overview/

## 版本说明

| 组件       | 版本号           |      | 组件           | 版本号       |
| ---------- | ---------------- | ---- | -------------- | ------------ |
| **helm chart** | **2.13.0** |      | **apiVersion** | v2    |
| **apisix** | **3.15.0-ubuntu** |      | **appVersion** | 3.15.0  |
| **apisix-ingress-controller** | **2.0.1** |      |  |   |

## 下载 helm chart

```properties
# 1. 先添加正确的官方仓库
helm repo add apisix https://charts.apiseven.com
# 2. 更新仓库信息
helm repo update
# 3. 搜索 apisix chart 的所有版本
helm search repo apisix/apisix --versions

wget https://github.com/apache/apisix-helm-chart/releases/download/apisix-2.13.0/apisix-2.13.0.tgz
tar -zxf apisix-2.13.0.tgz
```

----
## vim values.yaml

-  cd /k0s/apisix/helm-apisix

```properties
image:
  repository: 10.10.10.102:5000/apache/apisix
replicaCount: 5
service:
  type: LoadBalancer
apisix:
  enableIPv6: false
  ssl:
    enabled: true
    enableHTTP3: true
  deployment:
    mode: traditional
    role: "traditional"
    role_traditional:
      config_provider: "yaml"
  credentials:
    admin: 55f955be85405f2a9a8ccf1d90c08d08
    viewer: 8ab71a78d3c6514cba027bcf1319588b
  admin:
    allow:
      ipList: []
  nginx:
    keepaliveTimeout: 120s
etcd:
  enabled: false
ingress-controller:
  enabled: true
```

- **apisix.admin.allow.ipList: ["10.22.100.12/8"] ,  **置空测试
- **apisix.ssl.enabled=true**
- **credentials 更改同时更改如下 controller 的 adminKey **
- **apisix.nginx.keepaliveTimeout 修改120s 防止apisix ADC Server sync failed: socket hang up** 

----
## vim values.yaml  - ingress-controller

-  cd /k0s/apisix/helm-apisix/charts/apisix-ingress-controller

```properties
deployment:
  replicas: 3
  image:
    repository: 10.10.10.102:5000/apache/apisix-ingress-controller
  adcContainer:
    image:
      repository: 10.10.10.102:5000/api7/adc
      tag: "0.24.2"
    config:
      logLevel: "info"
config:
  logLevel: "info"
  provider:
    type: "apisix-standalone"
gatewayProxy:
  createDefault: true
  provider:
    controlPlane:
      auth:
        adminKey:
          value: "55f955be85405f2a9a8ccf1d90c08d08"
apisix:
  adminService:
    namespace: ingress-apisix
```



## 一键部署apisix 

```properties
kubectl create ns ingress-apisix
# 本地安装
cd /k0s/apisix/helm-apisix
helm -n ingress-apisix install apisix /k0s/apisix/helm-apisix -f values.yaml --wait
#-----输出----------------------------------------------------------------------------
NAME: apisix
LAST DEPLOYED: Thu Apr  2 11:45:03 2026
NAMESPACE: ingress-apisix
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
# 等待 部署 LoadBalancer  
kubectl -n ingress-apisix get svc apisix-gateway -w
```

## 检查校验

```properties
kubectl -n ingress-apisix get gatewayproxy apisix-config -o yaml
kubectl -n ingress-apisix get po -owide
kubectl -n ingress-apisix get svc
kubectl -n ingress-apisix port-forward svc/apisix-gateway 9080:80
# IngressClass = apisix
kubectl -n ingress-apisix get IngressClass apisix -oyaml
kubectl logs -n ingress-apisix apisix-ingress-controller-59ddcf9c8-nv55j -c adc-server
```



## 删除卸载

```properties
helm -n ingress-apisix uninstall apisix 
kubectl delete ns ingress-apisix
```



---

# 二、手动管理证书

官网：https://apisix-website-static.apiseven.com/zh/docs/ingress-controller/1.8.0/tutorials/mtls/

### 1. 创建 Secret

```properties
cd /k0s/apisix/ssl/develop.ekemp.com.cn
kubectl -n kubernetes-dashboard create secret generic ekemp-tls-secret --from-file=cert=tls.crt --from-file=key=tls.key
```
### 2. 创建 ApisixTls 资源

- cd /k0s/apisix/ssl

```properties
apiVersion: apisix.apache.org/v2
kind: ApisixTls
metadata:
  name: ekemp-tls
  namespace: kubernetes-dashboard
spec:
  ingressClassName: apisix 
  hosts:
    - k8s.develop.ekemp.com.cn
  secret:
    name: ekemp-tls-secret
    namespace: kubernetes-dashboard
```

- `hosts` 中的域名必须与你后续在路由中配置的域名严格一致，否则 TLS 握手会失败。
- ingressClassName: apisix  必须配置与 IngressClass = apisix 一致

### 3. 创建 ApisixRoute（关联路由）

- /k0s/apisix/ssl

```properties
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: httpserver-route
  namespace: kubernetes-dashboard
spec:
  ingressClassName: apisix
  http:
    - name: httpbin
      match:
        hosts:
          - k8s.develop.ekemp.com.cn 
        paths:
          - "/*"
      backends:
        - serviceName: httpbin
          servicePort: 8000
```

### 4. 验证 TLS 配置状态

```properties
kubectl get apisixtls ekemp-tls -o yaml
```

- 检查 `status` 字段是否为 `ResourcesAvailable` 和 `ResourcesSynced`。

### 验证 HTTPS 是否生效

```properties
curl -k --resolve "k8s.develop.ekemp.com.cn:443:10.10.10.130" https://k8s.develop.ekemp.com.cn/get?show_env=true  -vvv
```



### Apisix API

https://docs.api7.ai/apisix/reference/api-standalone-usage

```properties
ADMIN_API_KEY="55f955be85405f2a9a8ccf1d90c08d08"

curl http://apisix-admin.ingress-apisix:9180/apisix/admin/configs?format=json   -H "X-API-KEY: ${ADMIN_API_KEY}"
```







---

# 三、自动管理证书cert manager（待定）

官网：https://github.com/cert-manager/cert-manager

> cert-manager 将证书和证书颁发者作为 Kubernetes 集群中的资源类型添加进去，并简化了获取、续订和使用这些证书的过程。它支持从各种来源颁发证书，包括 Let's Encrypt (ACME)、HashiCorp Vault 和 CyberArk Certificate Manager，以及本地集群内颁发。cert-manager 还能确保证书保持有效和最新状态，在证书到期前适当的时候尝试续订证书，以降低中断的风险并减少工作量。






