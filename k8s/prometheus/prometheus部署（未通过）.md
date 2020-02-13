## 部署

在`110主机上`通过kubectl来建立Prometheus 需要的元件：

```bash
kubectl apply -f /k8s/prometheus/zhanggz/
kubectl apply -f /k8s/prometheus/zhanggz/operator/

# 这边要等 operator 起來并建立好 CRDs 才能进行
kubectl apply -f /k8s/prometheus/zhanggz/alertmanater/
kubectl apply -f /k8s/prometheus/zhanggz/node-exporter/
kubectl apply -f /k8s/prometheus/zhanggz/kube-state-metrics/
kubectl apply -f /k8s/prometheus/zhanggz/grafana/
kubectl apply -f /k8s/prometheus/zhanggz/kube-service-discovery/
```

## 补全ep信息

prometheus会收集存储metrics,grafana来视图输出
k8s的管理组件都会有metrics信息输出的端口,例如访问`curl localhost:10252/metrics`可以看到controller的metrics信息

​      上面我们知道如果prometheus监控管理组件的metrics需要(创建)ServiceMonitor选中一个svc,operator会查询到svc的ep(不需要svc有clusterip),根据配置的path从目标服务http的metrics页面拉取metrics信息
查看`ServiceMonitor`的yaml目录

```bash
cd /k8s/prometheus/zhanggz
ll servicemonitor/kube-*
...下面是输出
-rw-r--r-- 1 root root 567 Dec 23 20:20 servicemonitor/kube-apiserver-sm.yml
-rw-r--r-- 1 root root 374 Dec 23 20:20 servicemonitor/kube-controller-manager-sm.yml
-rw-r--r-- 1 root root 347 Dec 23 20:20 servicemonitor/kube-scheduler-sm.yml
```

 apiserver这个默认会在集群创建svc和生成ep(在default这个namespace里),无需理会

查看上面俩servicemonitor的yaml发现是选的kube-system(namespacs下)的标签为`k8s-app: kube-scheduler`和`k8s-app: kube-controller-manager`的svc
在下面目录找到官方单独创建了他俩的svc

```bash
$ ll kube-service-discovery/
total 8
-rw-r--r-- 1 root root 343 Dec 23 20:20 kube-controller-manager-svc.yml
-rw-r--r-- 1 root root 316 Dec 23 20:20 kube-scheduler-svc.yml
```

但是二进制跑的话由于不是pod不具有label属性,不会被上面俩svc选中,也就是俩svc的ep没创建需要我们手动创建
这种在需要把集群外的服务映射进来的时候就是创建一个同名的svc和ep,上面俩svc名字为

```bash
$ grep -P '^\s+name:' kube-service-discovery/kube-*
kube-service-discovery/kube-controller-manager-svc.yml:  name: kube-controller-manager-prometheus-discovery
kube-service-discovery/kube-scheduler-svc.yml:  name: kube-scheduler-prometheus-discovery
```

实际上默认集群带了组件同名的ep,但是operator为了不干预原有的,所以才用的上面的svc名字,同理operator也创建出了同名的ep,但是这俩ep是空的,需要我们手动修改

```
$ kubectl -n kube-system get ep 
NAME                                           ENDPOINTS                                                                    AGE
kube-controller-manager                        <none>                                                                       5m
kube-controller-manager-prometheus-discovery   <none>                                                                       5m
kube-dns                                       10.244.1.2:53,10.244.8.2:53,10.244.1.2:53 + 1 more...                        5m
kube-scheduler                                 <none>                                                                       5m
kube-scheduler-prometheus-discovery            <none>                                                                       5m
kubelet                                        192.168.88.111:10255,192.168.88.112:10255,192.168.88.113:10255 + 1 more...   5m
metrics-server                                 10.244.7.3:443         
```

可以看到为none,这里需要我们手动修改,下面为controller的信息注入,port是10252,servicemonitor里port名字为`https-metrics`,所以不要修改下面的port的name

同理注入kube-scheduler-prometheus-discovery的ep

创建EP,  vim ep.yaml

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager-prometheus-discovery
  namespace: kube-system
subsets:
- addresses:
  - ip: 192.168.10.110
  - ip: 192.168.10.111
  - ip: 192.168.10.120
  ports:
  - name: http-metrics
    port: 10252
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler-prometheus-discovery
  namespace: kube-system
subsets:
- addresses:
  - ip: 192.168.10.110
  - ip: 192.168.10.111
  - ip: 192.168.10.120
  ports:
  - name: http-metrics
    port: 10259
    protocol: TCP
```

最后执行

```bash
kubectl apply -f /k8s/prometheus/zhanggz/prometheus/
kubectl apply -f /k8s/prometheus/zhanggz/servicemonitor/
```

