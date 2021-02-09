## 下载官方文档



```bash
git clone https://github.com/coreos/prometheus-operator.git

# 新的下载地址 
git clone https://github.com/coreos/kube-prometheus.git
```

- master分支--当前发布版本v0.29.0 ，2019-10-18更改为v0.30.0, 解决prometheus-config-reloader cpu不足问题，在 0prometheus-operator-deployment.yaml 中设置 `--config-reloader-cpu=200m` 设置 200m即可。

```bash
cd contrib/kube-prometheus/manifests/
mkdir -p operator node-exporter alertmanager grafana kube-state-metrics prometheus serviceMonitor adapter
mv *-serviceMonitor* serviceMonitor/
mv 0prometheus-operator* operator/
mv grafana-* grafana/
mv kube-state-metrics-* kube-state-metrics/
mv alertmanager-* alertmanager/
mv node-exporter-* node-exporter/
mv prometheus-adapter* adapter/
mv prometheus-* prometheus/
$ ll
total 40
-rw-r--r-- 1 root root   60 Jan  6 14:15 00namespace-namespace.yaml
drwxr-xr-x 3 root root 4096 Jan  6 14:19 adapter/
drwxr-xr-x 3 root root 4096 Jan  6 14:19 alertmanager/
drwxr-xr-x 2 root root 4096 Jan  6 14:17 grafana/
drwxr-xr-x 2 root root 4096 Jan  6 14:17 kube-state-metrics/
drwxr-xr-x 2 root root 4096 Jan  6 14:18 node-exporter/
drwxr-xr-x 2 root root 4096 Jan  6 14:17 operator/
drwxr-xr-x 2 root root 4096 Jan  6 14:19 prometheus/
drwxr-xr-x 2 root root 4096 Jan  6 14:17 serviceMonitor/
```

## 为每个deploy,statefulset增加tolerations

kind: Alertmanager Prometheus, 直接加tolerations即可

[参考官方prometheus-operator API Docs](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#servicemonitor)

## 部署operator

先创建ns和operator,quay.io仓库拉取慢,可以使用我脚本拉取,其他镜像也可以这样去拉,不过在apply之前才能拉,一旦被docker接手拉取就只能漫长等

确认状态运行正常再往后执行,这里镜像是quay.io仓库的可能会很慢耐心等待或者自行修改成能拉取到的

```bash
$ kubectl -n monitoring get pod
NAME                                   READY     STATUS    RESTARTS   AGE
prometheus-operator-56954c76b5-qm9ww   1/1       Running   0          24s
```

## 更改kube-state-metrics-deployment.yaml的addon-resizer部分
因为addon-resizer2.1版本有pod pending[问题](https://github.com/kubernetes/kube-state-metrics/issues/672)，所以修改版本为1.8.4
```bash
- command:
  - /pod_nanny
  - --container=kube-state-metrics
  - --cpu=100m
  - --extra-cpu=1m
  - --memory=100Mi
  - --extra-memory=2Mi
  - --threshold=5
......
image: marvinpan/addon-resizer:1.8.4
name: addon-resizer
resources:
  limits:
    cpu: 150m
    memory: 50Mi
  requests:
    cpu: 150m
    memory: 50Mi
```
## (取消部署)`prometheus-adapter-apiService.yaml`
[参考官网GitHub修改](https://github.com/coreos/kube-prometheus/blob/master/experimental/custom-metrics-api/custom-metrics-apiservice.yaml)

移除prometheus-adapter-apiService.yaml，避免与`metrics-server`重复

自定义监控部署，请参考

- 此[GitHUb](<https://github.com/DirectXMan12/k8s-prometheus-adapter/tree/v0.4.1>)
- [此文](<https://segmentfault.com/a/1190000018141551?utm_source=tag-newest>),  [此GitHub](<https://github.com/stefanprodan/k8s-prom-hpa>)

## 部署整套CRD

创建相关的CRD,这里镜像可能也要很久,替换镜像名

```bash
kubectl apply -f adapter/
kubectl apply -f alertmanager/
kubectl apply -f node-exporter/
kubectl apply -f kube-state-metrics/
kubectl apply -f grafana/
---------------------------------
kubectl apply -f prometheus/
kubectl apply -f serviceMonitor/
```

查看CRD

```bash
[root@vmsrv-010-110 ~ (*|kubernetes:monitoring)]#  kubectl get crd
NAME                                    CREATED AT
alertmanagers.monitoring.coreos.com     2019-03-18T05:34:40Z
prometheuses.monitoring.coreos.com      2019-03-18T05:34:40Z
prometheusrules.monitoring.coreos.com   2019-03-18T05:34:40Z
servicemonitors.monitoring.coreos.com   2019-03-18T05:34:40Z
```

### grafana部署

作为prometheus前端展示页面，Grafana提供了强大的数据聚合和展示的功能，可以通过自定义前端配置修改dashboard，官方社区有很多[kubernetes的前端json文件](https://grafana.com/dashboards?search=kubernetes)供使用，

- `8588`-*Kubernetes Deployment Statefulset Daemonset metrics*
- `8919`-*Node Exporter 0.16 0.17 for Prometheus 监控展示看板*
- `11559`- Node Dashboard for Prometheus 中文版
- `10551`-NameSpace Based on Memory
- `6879`-Analysis by Pod
- `5851`  `4475`---traefik，5851优化后参考110-k8s文件夹
- `3070`-`9733`--`9618`---ETCD
- `7424`--kong
- `11049`---Flink
- `10041`- `8376`---glusterfs。10041优化后参考110机器文档

vim grafana-admin-secret.yml

```bash
apiVersion: v1
data:
  # jrtz | base64
  user: anJ0eg==
  # jrtz0755 | base64
  password: anJ0ejA3NTU=
kind: Secret
metadata:
  name: grafana-credentials
  namespace: monitoring
type: Opaque
```

grafana-deployment.yaml 增加

```bash
        env:
        - name: GF_AUTH_BASIC_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_SECURITY_ADMIN_USER
          valueFrom:
            secretKeyRef:
              name: grafana-credentials
              key: user
        - name: GF_SECURITY_ADMIN_PASSWORD
          valueFrom:
            secretKeyRef:
              name: grafana-credentials
              key: password
```

prometheus-serviceMonitorKubeControllerManager.yaml

```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    metricRelabelings:
    - action: drop
      regex: etcd_(debugging|disk|request|server).*
      sourceLabels:
      - __name__
    port: http-metrics
    ## add https
    scheme: https
    tlsConfig:
      caFile: /k8s/kubernetes/ssl/ca.pem
      certFile: /k8s/kubernetes/ssl/kube-controller-manager.pem
      keyFile: /k8s/kubernetes/ssl/kube-controller-manager-key.pem
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-controller-manager
```

## logs

```bash
kubectl logs -f prometheus-k8s-0 prometheus -n monitoring
kubectl logs -f prometheus-k8s-0 prometheus-config-reloader -n monitoring
kubectl get secret prometheus-k8s -o json -n monitoring
```

# secret

----

[参考个人博客](https://www.troyying.xyz/index.php/operate/15.html)

**创建ETCD对应访问证书**
因为监控ETCD不像监控kubernetes组件是通过api-server或直接使用http访问组件的target，需要使用https双向证书验证，所以要先创建可访问ETCD的secret，供prometheus server使用：

```bash
kubectl -n monitoring create secret generic etcd-certs --from-file=/etc/cni/net.d/calico-tls/etcd-cert --from-file=/etc/cni/net.d/calico-tls/etcd-key --from-file=/etc/cni/net.d/calico-tls/etcd-ca
```

证书、私钥及ca证书是可访问ETCD的，路径是主机本地存储证书的目录。
在prometheus的yaml文件中挂载证书：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: k8s
  labels:
    prometheus: k8s
spec:
  replicas: 2
  secrets: 
  - etcd-certs #增加这一句
  version: v1.7.1
```

如果已创建，可以直接edit该对象.

**创建ETCD的ServiceMonitor**

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-k8s
  labels:
    k8s-app: etcd-k8s
spec:
  jobLabel: k8s-app
  endpoints:
  - port: api
    interval: 30s
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/etcd-ca
      certFile: /etc/prometheus/secrets/etcd-certs/etcd-cert
      keyFile: /etc/prometheus/secrets/etcd-certs/etcd-key
      #use insecureSkipVerify only if you cannot use a Subject Alternative Name
      insecureSkipVerify: true
      #serverName: ETCD_DNS_OR_ALTERNAME_
  selector:
    matchLabels:
      k8s-app: etcd
  namespaceSelector:
    matchNames:
    - monitoring
```

其中`tlsConfig`的文件位置是prometheus容器里面挂载证书的位置，不确定的话可以进入容器内部验证一下
当证书`serverName`和etcd中签发的不匹配可以使用`insecureSkipVerify: true`

#### 数据采集到后，可以在 grafana 中导入编号为 `3070` 的 dashboard，获取到 etcd 的监控图表。

---
## 创建 kube-controller & kube-scheduler访问证书

```bash
kubectl -n monitoring create secret generic k8s-admin-certs --from-file=/k8s/kubernetes/ssl/ca.pem --from-file=/k8s/kubernetes/ssl/admin.pem --from-file=/k8s/kubernetes/ssl/admin-key.pem
```
#### 修改prometheus-prometheus.yaml, 增加secrets属性
[参考官方prometheus-operator API Docs](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#servicemonitor)

```bash
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  labels:
    prometheus: k8s
  name: k8s
  namespace: monitoring
spec:
  alerting:
    alertmanagers:
    - name: alertmanager-main
      namespace: monitoring
      port: web
  baseImage: marvinpan/prometheus
  nodeSelector:
    beta.kubernetes.io/os: linux
  replicas: 2
  resources:
    requests:
      memory: 400Mi
  ruleSelector:
    matchLabels:
      prometheus: k8s
      role: alert-rules
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: v2.5.0
  ## add secrets: string []
  secrets:
  - k8s-admin-certs
```

#### 修改prometheus-serviceMonitorKubeControllerManager.yaml

同时在kube-controller-manager-svc.yml ep.yaml改端口即可

```bash
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    metricRelabelings:
    - action: drop
      regex: etcd_(debugging|disk|request|server).*
      sourceLabels:
      - __name__
    port: http-metrics
    #===================add https=======================
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/k8s-admin-certs/ca.pem
      certFile: /etc/prometheus/secrets/k8s-admin-certs/admin.pem
      keyFile: /etc/prometheus/secrets/k8s-admin-certs/admin-key.pem
      #use insecureSkipVerify only if you cannot use a Subject Alternative Name
      insecureSkipVerify: true 
    #===================add==============================
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: kube-controller-manager
```

### prometheus-serviceMonitorKubeScheduler.yaml 同上

----

## Alertmanger触发器

更新Alertmanger到钉钉等触发器文件, base64编码，步骤如下：

- 复制`alertmanager-config.yaml`内容到浏览器上[在线 base64编码](<http://tool.chinaz.com/Tools/Base64.aspx>)，编码后为一串字符串，不含换行符
- 复制替换编码后的一串字符串到 `alertmanager-secret.yaml`的中
- 执行更新secret

```bash
k apply -f /k8s/prometheus/manifests/alertmanager/alertmanager-secret.yaml 
k create cm alert-default-tmpl --from-file=default.tmpl
```
新增自定义模板
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Alertmanager
metadata:
  labels:
    alertmanager: main
  name: main
  namespace: monitoring
spec:
  baseImage: marvinpan/prometheus-alertmanager
  nodeSelector:
    beta.kubernetes.io/os: linux
    node-role.kubernetes.io/monitoring: "monitoring"
  tolerations:
    - key: "node-role.kubernetes.io/monitoring"
      operator: "Equal"
      value: "monitoring"
      effect: "NoSchedule"
  replicas: 3
  securityContext:
    fsGroup: 2000
    runAsNonRoot: true
    runAsUser: 1000
  serviceAccountName: alertmanager-main
  version: v0.16.0
  # 新增自定义报警模板
  configMaps: 
    - alert-default-tmpl
```

- 删除alertmanager-main-0 ，1，2的pod，自动重启。
```bash
k delete po alertmanager-main-X
```

- 校验config,  进入alertmanager-main-X容器，执行检查

```bash
/bin/amtool check-config /etc/alertmanager/config/alertmanager.yaml 
-------------------------
Checking 'alertmanager.yaml'  SUCCESS
Found:
 - global config
 - route
 - 0 inhibit rules
 - 5 receivers
 - 0 templates
```

### 总结更新模板脚本

110机器

```bash
cd /k8s/prometheus/manifests/alertmanager
git pull
k delete cm alert-default-tmpl -n monitoring
k create cm alert-default-tmpl --from-file=default.tmpl -n monitoring
k delete po alertmanager-main-0 -n monitoring
k delete po alertmanager-main-1 -n monitoring
k delete po alertmanager-main-2 -n monitoring
k get po -n monitoring|grep alert 
sleep 6
k logs -f alertmanager-main-0 -c alertmanager
```



## Q1：prometheus报警CPUThrottlingHigh

- 修改`node-exporter-daemonset.yaml``
  - ``kube-rbac-proxy`的`limits.cpu=20m => 120m`
  - `node-exporter`的`limits.cpu=250m => 800m`
- 修改`kube-state-metrics-deployment.yaml`

  - `kube-rbac-proxy-main`的`limits.cpu=20m => 120m`

  - `kube-rbac-proxy-self`的`limits.cpu=20m => 120m`
- **prometheus-k8s-0**的**prometheus-config-reloader**容器cpu默认为50m
  - 错误在于 prometheus-operator的源码https://github.com/coreos/prometheus-operator/blob/v0.29.0/pkg/prometheus/statefulset.go设置写死了`v1.ResourceCPU:    resource.MustParse("50m")`
  - 在后续版本中已经优化，在`0prometheus-operator-deployment.yaml`中，增加参数`--config-reloader-cpu=120m`
  - 可以暂时手动更改解决：`kubectl edit statefulset prometheus-k8s` 的 `resources.limits.cpu: 120m`
- **metrics-server-v0.3.1-54f7588d75-znskf**的**metrics-server**容器cpu默认为48m
  - 修改metrics-server-deployment.yaml文件中pod_nanny  --cpu=40m ===》--cpu=100m，基础CPU调大，另外每个节点增加0.5m，所以总的限制CPU为108mi