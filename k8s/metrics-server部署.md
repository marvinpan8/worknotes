

# 部署 metrics-server 插件

Metrics-server是用来替换heapster获取集群上资源指标数据的，heapster从1.11开始逐渐被废弃了。

## 创建 metrics-server 使用的证书

配置 aggregator  ca证书文件

```bash
cd /jrtz/k8s/ssl/metrics
cat << EOF | tee ca-csr.json
{
    "CN": "aggregator",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Shenzhen",
            "ST": "Guangdong",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```bash
cat << EOF | tee ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
```

生成 aggregator  ca证书

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca 
```

创建 metrics-server 证书签名请求:

``` bash
cd /jrtz/k8s/ssl/metrics
cat <<EOF|tee metrics-server-csr.json 
{
  "CN": "aggregator",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "L": "Shenzhen",
      "ST": "Guangdong",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
+ 注意： CN 名称为 aggregator，需要与 kube-apiserver 的 --requestheader-allowed-names 参数配置一致；

+ ```javascript
  "CN"：Common Name，api-service 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
  "O"：Organization，api-service  从证书中提取该字段作为请求用户所属的组 (Group)；
  这两个参数在后面的kubernetes启用RBAC模式中很重要，因为需要设置kubelet、admin等角色权限，那么在配置证书的时候就必须配置对了，具体后面在部署kubernetes的时候会进行讲解。
  ```

生成 metrics-server 证书和私钥：

``` bash
cd /jrtz/k8s/ssl
cfssl gencert -ca=./ca.pem \
  -ca-key=./ca-key.pem  \
  -config=./ca-config.json  \
  -profile=kubernetes metrics-server-csr.json | cfssljson -bare metrics-server
```

将生成的证书和私钥文件拷贝到 kube-apiserver 节点：

``` bash
cp metrics-server*.pem /k8s/kubernetes/ssl/
NODE_IPS=(192.168.10.111 192.168.10.120)
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp metrics-server*.pem root@${node_ip}:/k8s/kubernetes/ssl/
  done
```

## 修改 kubernetes 控制平面组件的配置以支持 metrics-server

### kube-apiserver

添加如下配置参数：

``` bash
--requestheader-client-ca-file=/k8s/etcd/ssl/ca.pem \
--requestheader-allowed-names=aggregator \
--requestheader-extra-headers-prefix=X-Remote-Extra- \
--requestheader-group-headers=X-Remote-Group \
--requestheader-username-headers=X-Remote-User \
--proxy-client-cert-file=/k8s/etcd/ssl/metrics-server.pem \
--proxy-client-key-file=/k8s/etcd/ssl/metrics-server-key.pem \
--runtime-config=api/all=true \
```
+ `--requestheader-XXX`、`--proxy-client-XXX` 是 kube-apiserver 的 aggregator layer 相关的配置参数，metrics-server & HPA 需要使用；
+ `--requestheader-client-ca-file`：用于签名 `--proxy-client-cert-file` 和 `--proxy-client-key-file` 指定的证书；在启用了 metric aggregator 时使用；
+ 如果 --requestheader-allowed-names 不为空，则--proxy-client-cert-file 证书的 CN 必须位于 allowed-names 中，默认为 aggregator;( List of client certificate common names to allow to provide usernames in headers specified by --requestheader-username-headers.  **If empty, any client certificate validated by the authorities in --requestheader-client-ca-file is allowed.**)
+ `--enable-aggregator-routing`: Turns on aggregator routing requests to endpoints IP rather than cluster IP.(打开聚合器路由请求到端点IP，而不是集群IP。)
+ proxy-client-cert-file:  用于证明聚合器(aggregator)或kube-apiserver在请求期间发出呼叫的身份的客户端证书
+ proxy-client-key-file:  用于证明聚合器(aggregator)或kube-apiserver的身份的客户端证书的私钥，当它必须在请求期间调用时使用。包括将请求代理给用户api-server和调用webhook admission插件

> 如果 kube-apiserver 机器没有运行 kube-proxy，则还需要添加 `--enable-aggregator-routing=true` 参数；
>
> 关于 `--requestheader-XXX` 相关参数，参考：
>
> + https://github.com/kubernetes-incubator/apiserver-builder/blob/master/docs/concepts/auth.md
> + https://docs.bitnami.com/kubernetes/how-to/configure-autoscaling-custom-metrics/
>
> **注意：**requestheader-client-ca-file 指定的 CA 证书，必须具有 client auth and server auth；

### kube-controllr-manager

> 添加如下配置参数(从 v1.12 开始，该选项默认为 true，不需要再添加)：
>
>   --horizontal-pod-autoscaler-use-rest-clients=true
>
> 用于配置 HPA 控制器使用 REST 客户端获取 metrics 数据。
>
> --horizontal-pod-autoscaler-use-rest-clients has been deprecated, Heapster is no longer supported as a source for Horizontal Pod Autoscaler metrics.

## 修改插件配置文件

metrics-server 插件位于 kubernetes-src 的 `cluster/addons/metrics-server/` 目录下。

### 修改 metrics-server-deployment 文件：

- 修改镜像`marvinpan/metrics-server-amd64:v0.3.1` `marvinpan/addon-resizer:1.8.4`
- 注释 kubelet-port=10255（默认10250）
- 注释`deprecated-kubelet-completely-insecure`: 这很少是正确的选择，因为它使得kubelet通信完全不安全。如果您遇到身份验证错误，请确保您已在Kubelet上启用令牌webhook身份验证，如果您在具有自签名Kubelet证书的测试群集中，请考虑使用kubelet-insecure-tls。
- 增加`--kubelet-insecure-tls` : 因为10250是https端口，连接它时需要提供证书，所以加上，表示不验证客户端证书，此前的版本中使用--source=这个参数来指定不验证客户端证书。（不建议用于生产用途，但在具有自签名Kubelet服务证书的测试群集中非常有用。）
- 增加`--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname,InternalDNS,ExternalDNS `: 在确定用于连接到特定节点的地址时要使用的节点地址类型的优先级

+ `Kubeconfig`:  用于连接API-server和Kubelet的kubeconfig的路径（defaults to in-cluster config)

### 修改 addon-resizer
```bash
Usage of pod_nanny:
      --config-dir="": The name of directory used to specify resources for scaled container.
      --container="pod-nanny": The name of the container to watch. This defaults to the nanny itself.
      --cpu="MISSING": The base CPU resource requirement.
      --deployment="": The name of the deployment being monitored. This is required.
      --extra-cpu="0": The amount of CPU to add per node.
      --extra-memory="0Mi": The amount of memory to add per node.
      --extra-storage="0Gi": The amount of storage to add per node.
      --log-flush-frequency=5s: Maximum number of seconds between log flushes
      --memory="MISSING": The base memory resource requirement.
      --namespace=$MY_POD_NAMESPACE: The namespace of the ward. This defaults to the nanny's own pod.
      --pod=$MY_POD_NAME: The name of the pod to watch. This defaults to the nanny's own pod.
      --poll-period=10000: The time, in milliseconds, to poll the dependent container.
      --storage="MISSING": The base storage resource requirement.
      --threshold=0: A number between 0-100. The dependent's resources are rewritten when they deviate from expected by more than threshold.
      --estimator="linear", "The estimator to use. Currently supported: linear, exponential"
	  --minClusterSize=16, "The smallest number of nodes resources will be scaled to. Must be > 1. This flag is used only when an exponential estimator is used."
```

参考kubernetes-src\cluster\gce\gci\configure-helper.sh， if [[ "${ENABLE_SYSTEM_ADDON_RESOURCE_OPTIMIZATIONS:-}" == "true" ]]; then

```bash
base_metrics_server_cpu="40m"
base_metrics_server_memory="35Mi"
metrics_server_memory_per_node="4"
metrics_server_min_cluster_size="5"
```

### 修改resource-reader.yaml

```bash
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats    #新增这一行
  - namespaces
  verbs:
  - get
  - list
  - watch
```

#### 授予 kube-system:metrics-server ServiceAccount 访问 kubelet API 的权限(未使用)

``` bash
# 未使用
$ cat << EOF|tee auth-kubelet.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:kubelet-api-admin
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubelet-api-admin
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
EOF
```
+ 新建一个 ClusterRoleBindings 定义文件，授予相关权限；

## 创建 metrics-server

``` bash
$ pwd
/opt/k8s/kubernetes/cluster/addons/metrics-server
$ ls -l *.yaml
-rw-rw-r-- 1 k8s k8s  398 Jun  5 07:17 auth-delegator.yaml
-rw-rw-r-- 1 k8s k8s  404 Jun 16 18:02 auth-kubelet.yaml
-rw-rw-r-- 1 k8s k8s  419 Jun  5 07:17 auth-reader.yaml
-rw-rw-r-- 1 k8s k8s  393 Jun  5 07:17 metrics-apiservice.yaml
-rw-rw-r-- 1 k8s k8s 2640 Jun 16 17:54 metrics-server-deployment.yaml
-rw-rw-r-- 1 k8s k8s  336 Jun  5 07:17 metrics-server-service.yaml
-rw-rw-r-- 1 k8s k8s  801 Jun  5 07:17 resource-reader.yaml
$ kubectl create -f .
```

## 查看运行情况

``` bash
$ kubectl get pods -n kube-system |grep metrics-server
metrics-server-v0.2.1-7486f5bd67-v95q2   2/2       Running   0          45s

$ kubectl get svc -n kube-system|grep metrics-server
metrics-server         ClusterIP   10.254.115.120   <none>        443/TCP         1m
```

## 验证metrics-server

### 查看api

```bash
#metrics-server所提供的API
$ kubectl api-versions
...
autoscaling/v1
autoscaling/v2beta1
autoscaling/v2beta2
...
metrics.k8s.io/v1beta1
```

### 查看Metrics API数据

```bash
#启动一个代理以便curl api
root@k8s-master:~# kubectl proxy --port=8091
Starting to serve on 127.0.0.1:8091

#直接查看接口数据：
#可获取的资源：nodes和pods
root@k8s-master:~# curl localhost:8091/apis/metrics.k8s.io/v1beta1
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "nodes",
      "singularName": "",
      "namespaced": false,
      "kind": "NodeMetrics",
      "verbs": [
        "get",
        "list"
      ]
    },
    {
      "name": "pods",
      "singularName": "",
      "namespaced": true,
      "kind": "PodMetrics",
      "verbs": [
        "get",
        "list"
      ]
    }
  ]
```

```bash
#查看获取到的Node资源指标数据：cpu和内存
root@k8s-master:~# curl localhost:8091/apis/metrics.k8s.io/v1beta1/nodes
{
  "kind": "NodeMetricsList",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes"
  },
  "items": [
    {
      "metadata": {
        "name": "k8s-node02",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/k8s-node02",
        "creationTimestamp": "2018-10-06T08:48:51Z"
      },
      "timestamp": "2018-10-06T08:48:23Z",
      "window": "30s",
      "usage": {
        "cpu": "118493217n",
        "memory": "1320848Ki"
      }
```

kubectl top查看,  通过接口查看到了数据

```bash
root@k8s-master:/data/k8s/metrics-server# kubectl top node 
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master   323m         8%     2319Mi          40%       
k8s-node01   112m         2%     1327Mi          34%       
k8s-node02   114m         2%     1273Mi          33%  
```

## 查看 metrcs-server 输出的 metrics

metrics-server 输出的 APIs：https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md

1. 通过 kube-apiserver 或 kubectl proxy 访问：

    https://192.168.10.99:8443/apis/metrics.k8s.io/v1beta1/nodes
    https://192.168.10.99:8443/apis/metrics.k8s.io/v1beta1/nodes/{nodename}
    https://192.168.10.99:8443/apis/metrics.k8s.io/v1beta1/pods
    https://192.168.10.99:8443/apis/metrics.k8s.io/v1beta1/namespaces/{namespace}/pods

    https://192.168.10.99:8443/apis/metrics.k8s.io/v1beta1/namespaces/{namespace}/pods/{pod}

1. 直接使用 kubectl 命令访问：

    kubectl get --raw apis/metrics.k8s.io/v1beta1/nodes
    kubectl get --raw apis/metrics.k8s.io/v1beta1/pods
    kubectl get --raw apis/metrics.k8s.io/v1beta1/nodes/{nodename}
    kubectl get --raw apis/metrics.k8s.io/v1beta1/namespaces/{namespace}/pods/{pod}

``` bash
$ kubectl get --raw "/apis/metrics.k8s.io/v1beta1" | jq .
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "metrics.k8s.io/v1beta1",
  "resources": [
    {
      "name": "nodes",
      "singularName": "",
      "namespaced": false,
      "kind": "NodeMetrics",
      "verbs": [
        "get",
        "list"
      ]
    },
    {
      "name": "pods",
      "singularName": "",
      "namespaced": true,
      "kind": "PodMetrics",
      "verbs": [
        "get",
        "list"
      ]
    }
  ]
}

$ kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
{
  "kind": "NodeMetricsList",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes"
  },
  "items": [
    {
      "metadata": {
        "name": "m7-autocv-gpu03",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/m7-autocv-gpu03",
        "creationTimestamp": "2018-06-16T10:24:03Z"
      },
      "timestamp": "2018-06-16T10:23:00Z",
      "window": "1m0s",
      "usage": {
        "cpu": "133m",
        "memory": "1115728Ki"
      }
    },
    {
      "metadata": {
        "name": "m7-autocv-gpu01",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/m7-autocv-gpu01",
        "creationTimestamp": "2018-06-16T10:24:03Z"
      },
      "timestamp": "2018-06-16T10:23:00Z",
      "window": "1m0s",
      "usage": {
        "cpu": "221m",
        "memory": "6799908Ki"
      }
    },
    {
      "metadata": {
        "name": "m7-autocv-gpu02",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/m7-autocv-gpu02",
        "creationTimestamp": "2018-06-16T10:24:03Z"
      },
      "timestamp": "2018-06-16T10:23:00Z",
      "window": "1m0s",
      "usage": {
        "cpu": "76m",
        "memory": "1130180Ki"
      }
    }
  ]
}
```
+ /apis/metrics.k8s.io/v1beta1/nodes 和 /apis/metrics.k8s.io/v1beta1/pods 返回的 usage 包含 CPU 和 Memory；

## 参考：
1. https://kubernetes.feisky.xyz/zh/addons/metrics.html
1. metrics-server RBAC：https://github.com/kubernetes-incubator/metrics-server/issues/40
1. metrics-server 参数：https://github.com/kubernetes-incubator/metrics-server/issues/25
1. https://kubernetes.io/docs/tasks/debug-application-cluster/core-metrics-pipeline/

### Q: 在部署过程中遇到的错误：

```
Get https://k8s-master:10250/stats/summary/: dial tcp: lookup k8s-master on 10.96.0.10:53: no such host
```

提示 无法解析节点的主机名，是metrics-server这个容器不能通过CoreDNS 10.96.0.10:53 解析各Node的主机名，metrics-server连节点时默认是连接节点的主机名，需要加个参数，让它连接节点的IP：
“--kubelet-preferred-address-types=InternalIP”

