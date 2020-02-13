# 部署 kube-proxy 组件

kube-proxy 运行在所有 worker 节点上，，它监听 apiserver 中 service 和 Endpoint 的变化情况，创建路由规则来进行服务负载均衡。

本文档讲解部署 kube-proxy 的部署，使用 ipvs 模式。
## 下载和分发 kube-proxy 二进制文件

参考 [06-0.部署master节点.md](06-0.部署master节点.md)

## 安装依赖包

各节点需要安装 `ipvsadm` 和 `ipset` 命令，加载 `ip_vs` 内核模块。

参考 [07-0.部署worker节点.md](07-0.部署worker节点.md)

## 创建 kube-proxy 证书

创建证书签名请求：

``` bash
cd /jrtz/k8s/ssl
cat << EOF | tee kube-proxy-csr.json 
{
  "CN": "system:kube-proxy",
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
+ CN：指定该证书的 User 为 `system:kube-proxy`；
+ 预定义的 RoleBinding `system:node-proxier` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；
+ 该证书只会被 kube-proxy 当做 client 证书使用，所以 hosts 字段为空；

生成证书和私钥：

``` bash
cd /jrtz/k8s/ssl
cfssl gencert -ca=./ca.pem \
  -ca-key=./ca-key.pem \
  -config=./ca-config.json \
  -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
ls kube-proxy*
```
## 创建和分发 kubeconfig 文件

``` bash
cd /jrtz/k8s/ssl
export KUBE_APISERVER="https://192.168.10.99:8443"
kubectl config set-cluster kubernetes \
  --certificate-authority=/k8s/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
+ `--embed-certs=true`：将 ca.pem 和 admin.pem 证书内容嵌入到生成的 kubectl-proxy.kubeconfig 文件中(不加时，写入的是证书文件路径)；

分发 kubeconfig 文件：

``` bash
cd /jrtz/k8s/ssl
cp kube-proxy.kubeconfig /k8s/kubernetes/ssl/
export NODE_IPS=(192.168.10.112 192.168.10.121)
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    scp kube-proxy.kubeconfig root@${node_ip}:/k8s/kubernetes/ssl/
  done
```
## 创建 kube-proxy 配置文件

从 v1.10 开始，kube-proxy **部分参数**可以配置文件中配置。可以使用 `--write-config-to` 选项生成该配置文件，或者参考 [kubeproxyconfig 的类型定义源文件](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/apis/kubeproxyconfig/types.go)

创建 kube-proxy config 文件模板：

``` bash
cd /jrtz/k8s/ssl
# Pod 网段，建议 /16 段地址，部署前路由不可达，部署后集群内路由可达(calico/flanneld 保证)
CLUSTER_CIDR="172.30.0.0/16"
cat << EOF | tee kube-proxy-config.yaml.template
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/k8s/kubernetes/ssl/kube-proxy.kubeconfig"
bindAddress: ##NODE_IP##
clusterCIDR: ${CLUSTER_CIDR}
healthzBindAddress: ##NODE_IP##:10256
hostnameOverride: ##NODE_IP##
metricsBindAddress: ##NODE_IP##:10249
mode: "ipvs"
EOF
```
+ `bindAddress`: 监听地址；
+ `clientConnection.kubeconfig`: 连接 apiserver 的 kubeconfig 文件；
+ `clusterCIDR`: kube-proxy 根据 `--cluster-cidr` 判断集群内部和外部流量，指定 `--cluster-cidr` 或 `--masquerade-all` 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
+ `hostnameOverride`: **参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 ipvs 规则；**
+ `mode`: 使用 ipvs 模式；

为各节点创建和分发 kube-proxy 配置文件：

``` bash
cd /jrtz/k8s/ssl
export NODE_IPS=(192.168.10.112 192.168.10.121)
for node_ip in ${NODE_IPS[@]}
  do 
    echo ">>> ${node_ip}"
    sed -e "s/##NODE_IP##/${node_ip}/" kube-proxy-config.yaml.template > kube-proxy-config-${node_ip}.yaml.template
    scp kube-proxy-config-${node_ip}.yaml.template root@${node_ip}:/k8s/kubernetes/ssl/kube-proxy-config.yaml
    rm -f kube-proxy-config-${node_ip}.yaml.template
  done
```
## 创建和分发 kube-proxy systemd unit 文件

``` bash
cd /jrtz/k8s/cfg
export K8S_DIR="/data/k8s/k8s"
cat <<EOF |tee kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=${K8S_DIR}/kube-proxy
ExecStart=/k8s/kubernetes/bin/kube-proxy \\
  --config=/k8s/kubernetes/ssl/kube-proxy-config.yaml \\
  --logtostderr=true \\
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
分发 kube-proxy systemd unit 文件：

``` bash
cd /jrtz/k8s/cfg
cp kube-proxy.service /etc/systemd/system/
export NODE_IPS=(192.168.10.112 192.168.10.121)
for node_ip in ${NODE_IPS[@]}
  do 
    echo ">>> ${node_ip}"
    scp /jrtz/k8s/bin/kube-proxy root@${node_ip}:/k8s/kubernetes/bin/
    scp kube-proxy.service root@${node_ip}:/etc/systemd/system/
  done
```

## 启动 kube-proxy 服务

``` bash
cd /rtz/k8s/ssl
export K8S_DIR="/data/k8s/k8s"
export NODE_IPS=(192.168.10.112 192.168.10.121)
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "mkdir -p ${K8S_DIR}/kube-proxy"
    ssh root@${node_ip} "systemctl daemon-reload && systemctl enable kube-proxy && systemctl restart kube-proxy"
  done
```
+ 必须先创建工作目录；

## 检查启动结果

``` bash
cd /rtz/k8s/ssl
export NODE_IPS=(192.168.10.112 192.168.10.121)
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "systemctl status kube-proxy|grep Active"
  done
```

确保状态为 `active (running)`，否则查看日志，确认原因：

``` bash
journalctl -u kube-proxy -f -n 500
```

## 查看监听端口和 metrics

``` bash
$ sudo netstat -lnpt|grep kube-prox
tcp        0      0 172.27.128.150:10249    0.0.0.0:*               LISTEN      76419/kube-proxy
tcp        0      0 172.27.128.150:10256    0.0.0.0:*               LISTEN      76419/kube-proxy
```
+ 10249：http prometheus metrics port;
+ 10256：http healthz port;

## 查看 ipvs 路由规则

``` bash
cd /rtz/k8s/ssl
export NODE_IPS=(192.168.10.112 192.168.10.121)
for node_ip in ${NODE_IPS[@]}
  do
    echo ">>> ${node_ip}"
    ssh root@${node_ip} "/usr/sbin/ipvsadm -ln"
  done
```

预期输出：

``` bash
>>> 172.27.128.150
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 172.27.128.149:6443          Masq    1      0          0
  -> 172.27.128.148:6443          Masq    1      0          0
  -> 172.27.128.150:6443          Masq    1      0          0
>>> 172.27.128.149
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 172.27.128.149:6443          Masq    1      0          0
  -> 172.27.128.148:6443          Masq    1      0          0
  -> 172.27.128.150:6443          Masq    1      0          0
>>> 172.27.128.148
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 172.27.128.149:6443          Masq    1      0          0
  -> 172.27.128.148:6443          Masq    1      0          0
  -> 172.27.128.150:6443          Masq    1      0          0
```
可见将所有到 kubernetes cluster ip 443 端口的请求都转发到 kube-apiserver 的 6443 端口；