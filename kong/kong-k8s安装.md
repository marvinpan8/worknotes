## kong在k8s中安装

> 参考官方文档<https://github.com/Kong/kong-dist-kubernetes>

---

`kubectl create cm prometheus-cm --from-file=./prometheus-server.conf -n kong`

```bash
server {
    server_name kong_prometheus_exporter;
    listen 0.0.0.0:9542; 

    location / {
        default_type text/plain;
        content_by_lua_block {
            local promethus = require "kong.plugins.prometheus.exporter"
            promethus:collect()
        }
    }

    location /nginx_status {
        internal;
        access_log off;
        stub_status;
    }
}
```



#### kong的prometheus对接

prometheus-serviceMonitorKong.yaml
```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kong-ingress-data-plane
  name: kong-ingress-data-plane
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: prometheus
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kong
  selector:
    matchLabels:
      k8s-app: kong-ingress-data-plane
```

#### prometheus增加命名空间kong支持

prometheus-roleSpecificNamespaces.yaml 中增加

```bash
- apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: prometheus-k8s
    namespace: kong
  rules:
  - apiGroups:
    - ""
    resources:
    - services
    - endpoints
    - pods
    verbs:
    - get
    - list
    - watch
```

prometheus-roleBindingSpecificNamespaces.yaml 中增加

```bash
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: prometheus-k8s
    namespace: kong
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: prometheus-k8s
  subjects:
  - kind: ServiceAccount
    name: prometheus-k8s
    namespace: monitoring
```





