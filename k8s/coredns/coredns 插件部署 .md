# coredns 插件部署 

## 修改配置文件

将下载的 kubernetes-server-linux-amd64.tar.gz 解压后，再解压其中的 kubernetes-src.tar.gz 文件。

coredns 对应的目录是：`cluster/addons/dns`。

```bash
mkdir /k8s/coredns && cd /k8s/coredns
# 复制 coredns.yaml.base 到该目录下
mv coredns.yaml.base coredns.yaml
# 集群 DNS 域名（末尾不带点号）
export CLUSTER_DNS_DOMAIN="cluster.local"
# 集群 DNS 服务 IP (从 SERVICE_CIDR 中预分配)
export CLUSTER_DNS_SVC_IP="10.254.0.2"
sed -i -e "s/__PILLAR__DNS__DOMAIN__/${CLUSTER_DNS_DOMAIN}/" -e "s/__PILLAR__DNS__SERVER__/${CLUSTER_DNS_SVC_IP}/" coredns.yaml
sed -i -e "s/k8s.gcr.io/marvinpan/" coredns.yaml
```

### 增加90行注释部分 replicas: 2

## 创建 coredns

``` bash
kubectl create -f coredns.yaml
```

## 检查 coredns 功能

``` bash
$ kubectl get all -n kube-system
NAME                           READY     STATUS    RESTARTS   AGE
pod/coredns-77c989547b-6l6jr   1/1       Running   0          3m
pod/coredns-77c989547b-d9lts   1/1       Running   0          3m

NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
service/coredns   ClusterIP   10.254.0.2   <none>        53/UDP,53/TCP   3m

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   2         2         2            2           3m

NAME                                 DESIRED   CURRENT   READY     AGE
replicaset.apps/coredns-77c989547b   2         2         2         3m
```