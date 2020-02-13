# 系统要求

###  node要求
- AMD64 处理器
- linux内核kernel 3.10 以上
- CentOS 7
### Key/value 存储选择
- etcdv3 cluster（推荐）/Kubernetes API datastore（不推荐）
### 网络要求

Ensure that your hosts and firewalls allow the necessary traffic based on your configuration.

| Configuration                                     | Host(s)             | Connection type | Port/protocol                                                |
| ------------------------------------------------- | ------------------- | --------------- | ------------------------------------------------------------ |
| Calico networking (BGP)                           | All                 | Bidirectional   | TCP 179                                                      |
| Calico networking with IP-in-IP enabled (default) | All                 | Bidirectional   | IP-in-IP, often represented by its protocol number `4`       |
| Calico networking with Typha enabled              | Typha agent hosts   | Incoming        | TCP 5473 (default)                                           |
| flannel networking (VXLAN)                        | All                 | Bidirectional   | UDP 4789                                                     |
| All                                               | kube-apiserver host | Incoming        | Often TCP 443 or 6443*                                       |
| etcd datastore                                    | etcd hosts          | Incoming        | [Officially](http://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.txt) TCP 2379 but can vary |

\* *The value passed to kube-apiserver using the --secure-port flag. If you cannot locate this, check the targetPort value returned by kubectl get svc kubernetes -o yaml.*

## Kubernetes要求
- v1.10/v1.11/v1.12
- **kubecet 必须配置启动参数`--network-plugin=cni**`

- 不支持k8s集群网络迁移
- 支持的kube-proxy模式
  - `iptables` (default)
  - `ipvs` Requires Kubernetes >=v1.9.3. Refer to [Enabling IPVS in Kubernetes](https://docs.projectcalico.org/v3.3/usage/enabling-ipvs) for more details. 如果Calico检测到`kube-proxy`在该模式下运行，则会自动激活。

> `ipvs`模式提供比`iptables`模式更大的规模和性能。但是，它有一些限制。在IPVS模式下的功能如下：
>
> - Calico需要[额外的`iptables`数据包标记位](https://docs.projectcalico.org/v3.3/reference/felix/configuration#ipvs-bits) ，以便在数据包通过IPVS时对数据包进行跟踪。
> - Calico需要[配置](https://docs.projectcalico.org/v3.3/reference/felix/configuration#ipvs-portranges) （KubeNodePortRanges ---Default: `30000:32767`）分配给Kubernetes NodePorts的端口范围。如果服务确实使用Calico预期范围之外的NodePort，Calico会将这些端口的流量视为主机流量而不是pod流量。
> - Calico支持使用本地分配`ExternalIP`给Kubernetes服务。
>

## 配置NetworkManager

确保Calico可以管理主机上的cali和tunl接口。 如果主机上存在NetworkManager，请按照下面方法配置NetworkManager。

NetworkManager管理默认网络命名空间中接口的路由表的功能，可能会干扰Calico正确处理网络路由的能力。

创建一个配置文件/etc/NetworkManager/conf.d/calico.conf，来制止这种干扰：

```bash
vim /etc/NetworkManager/conf.d/calico.conf
---------------------------------------------
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*
```
------

通过Calico manifests yaml文件安装
======

### 安装资源:

- Installs the `calico/node` container on each host using a DaemonSet.
- Installs the Calico CNI binaries and network config on each host using a DaemonSet.
- Runs `calico/kube-controllers` as a deployment.
- The `calico-etcd-secrets` secret, which optionally allows for providing etcd TLS assets.
- The `calico-config` ConfigMap, which contains parameters for configuring the install.

###  修改calico.yaml配置

https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/calico.yaml

- `CALICO_IPV4POOL_CIDR`设置为 `172.30.0.0/16`
- `CALICO_IPV4POOL_IPIP`]设置 `Never`
- image分别设置为`marvinpan/calico-node:v3.3.1`,`marvinpan/calico-cni:v3.3.1`,`marvinpan/calico-kube-controllers:v3.3.1`

### Etcd 配置

By default, these manifests do not configure secure access to etcd and assume an etcd proxy is running on each host. The following configuration options let you specify custom etcd cluster endpoints as well as TLS.

The following table outlines the supported `ConfigMap` options for etcd:

| Option         | Description                                                  | Default               |
| -------------- | ------------------------------------------------------------ | --------------------- |
| etcd_endpoints | Comma-delimited list of etcd endpoints to connect to.        | http://127.0.0.1:2379 |
| etcd_ca        | The file containing the root certificate of the CA that issued the etcd server certificate. Configures `calico/node`, the CNI plugin, and the Kubernetes controllers to trust the signature on the certificates provided by the etcd server. | None                  |
| etcd_key       | The file containing the private key of the `calico/node`, the CNI plugin, and the Kubernetes controllers client certificate. Enables these components to participate in mutual TLS authentication and identify themselves to the etcd server. | None                  |
| etcd_cert      | The file containing the client certificate issued to `calico/node`, the CNI plugin, and the Kubernetes controllers. Enables these components to participate in mutual TLS authentication and identify themselves to the etcd server. | None                  |

To use these manifests with a TLS-enabled etcd cluster you must do the following:

1. 在`ConfigMap` 部分, 取消注释 `etcd_ca`, `etcd_key`, and `etcd_cert`，如下

   **注意**：文件路径和文件名保持不变

   ```bash
   etcd_ca: "/calico-secrets/etcd-ca"
   etcd_cert: "/calico-secrets/etcd-cert"
   etcd_key: "/calico-secrets/etcd-key"
   ```

2. 修改etcd_endpoints

   ```bash
   etcd_endpoints: "https://192.168.10.110:2379,https://192.168.10.111:2379,https://192.168.10.120:2379"
   ```

   

3. base64编码，`base64 -w 0`代表解析新行, 若-w无效，可以用`base64 | tr -d '\n'`

   ```
   cat <file> | base64 -w 0 
   ```

   In the `Secret` named `calico-etcd-secrets`, 取消注释`etcd_ca`, `etcd_key`, and `etcd_cert` and 粘贴恰当的base64编码值

   ```bash
   apiVersion: v1
   kind: Secret
   type: Opaque
   metadata:
     name: calico-etcd-secrets
     namespace: kube-system
   data:
     # Populate the following files with etcd TLS configuration if desired, but leave blank if
     # not using TLS for etcd.
     # This self-hosted install expects three files with the following names.  The values
     # should be base64 encoded strings of the entire contents of each file.
     etcd-key: "LS0tLS1CRUdJTiB...VZBVEUgS0VZLS0tLS0="
     etcd-cert: "LS0tLS1...ElGSUNBVEUtLS0tLQ=="
     etcd-ca: "LS0tLS1CRUdJTiBD...JRklDQVRFLS0tLS0="
   ```

4. 在`calico-kube-controllers`部分，增加nodeSelector

   `node-role.kubernetes.io/master: ''`

5. 配置 the roles and bindings

   ```
   kubectl apply -f \
   https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/rbac.yaml
   ```

6. 创建etcd  configmap,指定/k8s/etcd/ssl目录下的所有文件，包括ca.pem, etcd.pem,etcd-key.pem

   ```bash
   kubectl create configmap etcd-pem --from-file=/k8s/etcd/ssl -n kube-system
   ```

7. 所有机器创建根目录文件夹calico-secrets，并拷贝ca.pem, etcd.pem,etcd-key.pem到该文件夹下

   ```bash
   mkdir /calico-secrets
   cd /jrtz/etcd/ssl/
   cp ca.pem, etcd.pem,etcd-key.pem /calico-secrets/
   ```

8. 执行kubectl

   ```bash
   kubectl apply -f calico.yaml
   ```
### 权限授权（默认）
Calico’s manifests 指派了 1~2 service accounts. 依赖 cluster’s authorization mode, 最好备份 service accounts.

### 其他配置（默认）

The following table outlines the remaining supported `ConfigMap` options.

| Option             | Description                                                  | Default |
| ------------------ | ------------------------------------------------------------ | ------: |
| calico_backend     | The backend to use.                                          |  `bird` |
| cni_network_config | The CNI Network config to install on each node. Supports templating as described below.（如下） |         |

### 配置 cni_network_config template（默认）

The `cni_network_config` configuration option supports the following template fields, which will be filled in automatically by the `calico/cni` container:

| Field                         | Substituted with                                             |
| ----------------------------- | :----------------------------------------------------------- |
| `__KUBERNETES_SERVICE_HOST__` | The Kubernetes service Cluster IP, e.g `10.0.0.1`            |
| `__KUBERNETES_SERVICE_PORT__` | The Kubernetes service port, e.g., `443`                     |
| `__SERVICEACCOUNT_TOKEN__`    | The service account token for the namespace, if one exists.  |
| `__ETCD_ENDPOINTS__`          | The etcd endpoints specified in `etcd_endpoints`.            |
| `__KUBECONFIG_FILEPATH__`     | The path to the automatically generated kubeconfig file in the same directory as the CNI network configuration file. |
| `__ETCD_KEY_FILE__`           | The path to the etcd key file installed to the host. Empty if no key is present. |
| `__ETCD_CERT_FILE__`          | The path to the etcd certificate file installed to the host, empty if no cert present. |
| `__ETCD_CA_CERT_FILE__`       | The path to the etcd certificate authority file installed to the host. Empty if no certificate authority is present. |

## 检查测试

### 查看本地文件夹 

```bash
netstat -anp |grep calico
-----------------------------------------------------------
tcp     0      0 127.0.0.1:9099      0.0.0.0:*      LISTEN      3948/calico-node
-----------------------------------------------------------
cd /etc/cni/net.d/ && ll
-----------------------------------------------------------
-rw-rw-r-- 1 root root  758 Mar  8 11:19 10-calico.conflist
-rw------- 1 root root 2994 Mar  8 11:19 calico-kubeconfig
drwxr-xr-x 2 root root   54 Mar  8 11:19 calico-tls
-----------------------------------------------------------
查看10-calico.conflist， calico-kubeconfig calico-tls文件夹下有三认证文件
```



### copy calicoctl 到M1的/k8s/kubernetes/bin

```bash
$ calicoctl node status
-------------------------
Calico process is running.
IPv4 BGP status
+----------------+-------------------+-------+----------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+----------------+-------------------+-------+----------+-------------+
| 192.168.10.110 | node-to-node mesh | up    | 12:39:24 | Established |
| 192.168.10.112 | node-to-node mesh | up    | 12:39:23 | Established |
| 192.168.10.120 | node-to-node mesh | up    | 12:39:25 | Established |
| 192.168.10.121 | node-to-node mesh | up    | 12:39:31 | Established |
+----------------+-------------------+-------+----------+-------------+
IPv6 BGP status
No IPv6 peers found.
```

创建配置文件

```bash
cat << EOF | tee /etc/calico/calicoctl.cfg
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "etcdv3"
  etcdEndpoints: "https://192.168.10.110:2379,https://192.168.10.111:2379,https://192.168.10.120:2379"
  etcdKeyFile: /k8s/etcd/ssl/etcd-key.pem
  etcdCertFile: /k8s/etcd/ssl/etcd.pem
  etcdCACertFile: /k8s/etcd/ssl/ca.pem
EOF
```

测试命令

```bash
calicoctl get nodes
calicoctl get bgppeers
calicoctl get ippool
calicoctl get profiles -o wide
# 查看当前有哪些使用中的NetworkPolicy：
calicoctl get policy --all-namespaces
```

**在单个主机上将calicoctl作为容器安装**

docker pull quay.io/calico/ctl:v3.4.0

**将calicoctl作为Kubernetes pod安装**

- etcd作为后端datastore时, 如果启用了etcd TLS，则需要对下面配置文件更新证书和密钥部分

```bash
kubectl apply -f https://docs.projectcalico.org/v3.4/getting-started/kubernetes/installation/hosted/calicoctl.yaml
```

- Kubernetes API datastore时

```
kubectl apply -f https://docs.projectcalico.org/v3.4/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calicoctl.yaml
```

使用calicoctl命令的方法

```
kubectl exec -ti -n kube-system calicoctl — /calicoctl get profiles -o wide
```

推荐设置成一个alias别名

```
alias calicoctl="kubectl exec -i -n kube-system calicoctl /calicoctl — "
```

### 查看Calico插件日志的方法

Calico默认的日志输出级别为WARNING，输出到stderr，可以按需配置为INFO或DEBUG，日志由kubelet记录，如果是使用的systemd管理服务，则可以使用`journalctl -u kubelet -f -n 500`查看日志信息。

### 测试POD

```bash
$ kubectl create ns policy-demo
 # Run the Pods.
$ kubectl run --namespace=policy-demo nginx --replicas=2 --image=nginx
 # Create the Service.
$ kubectl expose --namespace=policy-demo deployment nginx --port=80
 # Run a Pod and try to access the `nginx` Service.
$ kubectl run --namespace=policy-demo access --rm -ti --image busybox /bin/sh
# Waiting for pod policy-demo/access-472357175-y0m47 to be running, status is Pending, pod ready: false
# If you don't see a command prompt, try pressing enter.进入容器
$ wget -q --timeout=5 nginx -O -
# 该nslookup命令可能需要一分钟或更长时间才能超时。
$ nslookup nginx
# 退出容器，删除namespace,清理所有
$ kubectl delete ns policy-demo
```





-----

# 二进制安装集成指南

## Calico 组件

Calico 与 Kubernetes集成有三个组件：

- 每节点安装 Docker容器 `calico/node`.包含路由发生器的BGP agent，编写网络策略规则的Felix agent .
- The [cni-plugin](https://github.com/projectcalico/cni-plugin) network plugin binaries. 有两个二进制可执行文件和一个配置文件。直接与每个节点上的Kubernetes 的`kubelet` 进程集成，以发现已创建的pod，并将其添加到Calico网络。
- The Calico Kubernetes controllers, 单个pod运行。这些组件监视Kubernetes API以保持Calico同步.

## 所有节点安装 calico/node

- [ ] Run calico/node and configure the node.

  See the [`calicoctl node run` documentation](https://docs.projectcalico.org/v3.3/reference/calicoctl/commands/node/) for more information.

  ```bash
  # Download and install calicoctl
  wget https://github.com/projectcalico/calicoctl/releases/download/v3.3.1/calicoctl
  sudo chmod +x calicoctl
  
  # Run the calico/node container
  sudo ETCD_ENDPOINTS=http://<ETCD_IP>:<ETCD_PORT> ./calicoctl node run --node-image=calico/node:v3.3.1
  ```

- [ ] Example systemd unit file (calico-node.service),替换<ETCD_IP>:<ETCD_PORT>，HOSTNAME

  ```bash
  [Unit]
  Description=calico-node
  After=docker.service
  Requires=docker.service
  
  [Service]
  User=root
  Environment=ETCD_ENDPOINTS=http://<ETCD_IP>:<ETCD_PORT>
  PermissionsStartOnly=true
  ExecStart=/usr/bin/docker run --net=host --privileged --name=calico-node \
    -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \
    -e NODENAME=${HOSTNAME} \
    -e CALICO_IPV4POOL_IPIP=Never \
    -e IP=autodetect \
    -e IP6= \
    -e AS= \
    -e NO_DEFAULT_POOLS= \
    -e CALICO_IPV4POOL_CIDR="172.30.0.0/16" \
    -e CALICO_LIBNETWORK_ENABLED=false \
    -e CALICO_NETWORKING_BACKEND=bird \
    -e FELIX_DEFAULTENDPOINTTOHOSTACTION=ACCEPT \
    -v /lib/modules:/lib/modules \
    -v /run/docker/plugins:/run/docker/plugins \
    -v /var/run/calico:/var/run/calico \
    -v /var/log/calico:/var/log/calico \
    -v /var/lib/calico:/var/lib/calico \
    calico/node:v3.3.1
  ExecStop=/usr/bin/docker rm -f calico-node
  Restart=always
  RestartSec=10
  
  [Install]
  WantedBy=multi-user.target
  ```

##  安装Calico CNI 插件

**Kubernetes 的`kubelet`必须配置支持`calico` and `calico-ipam`插件。 **

###  安装Calico插件

下载文件

```bash
wget -N https://github.com/projectcalico/cni-plugin/releases/download/v3.3.1/calico-amd64
wget -N https://github.com/projectcalico/cni-plugin/releases/download/v3.3.1/calico-ipam-amd64
mv ./calico-amd64 /opt/cni/bin/calico
mv ./calico-ipam-amd64 /opt/cni/bin/calico-ipam
chmod +x /opt/cni/bin/calico /opt/cni/bin/calico-ipam
```

需要一个配置文件，

**注意：**只有运行calico/kube-controllers才需要policy部分，唯一支持类型k8s，calico/kube-controllers必须enabled the policy, profile, and workloadendpoint controllers ,Calico CNI插件必须要有POD的API的只读权限

```bash
mkdir -p /etc/cni/net.d
cat <<EOF | tee /etc/cni/net.d/10-calico.conf 
{
    "name": "calico-k8s-network",
    "cniVersion": "0.6.0",
    "type": "calico",
    "etcd_endpoints": "https://192.168.10.110:2379,https://192.168.10.111:2379,https://192.168.10.120:2379",
    "etcd_key_file": "/k8s/etcd/ssl/etcd-key.pem",
    "etcd_cert_file": "/k8s/etcd/ssl/etcd.pem",
    "etcd_ca_cert_file": "/k8s/etcd/ssl/ca.pem",
    "log_level": "info",
    "ipam": {
        "type": "calico-ipam"
    },
    "policy": {
        "type": "k8s"
    },
    "kubernetes": {
        "kubeconfig": "</PATH/TO/KUBECONFIG>"
    }
}
EOF
```

Replace `<ETCD_IP>:<ETCD_PORT>` with your etcd configuration. Replace `</PATH/TO/KUBECONFIG>` with your kubeconfig file. See [Kubernetes kubeconfig](http://kubernetes.io/docs/user-guide/kubeconfig-file/) for more information about kubeconfig.

支持指定IP，IP范围，For more information on configuring the Calico CNI plugins, see the [configuration guide](https://docs.projectcalico.org/v3.3/reference/cni-plugin/configuration)

### 安装标准的 CNI loopback 插件

```bash
wget https://github.com/containernetworking/plugins/releases/download/v0.7.1/cni-plugins-amd64-v0.7.1.tgz
tar -zxvf cni-plugins-amd64-v0.7.1.tgz
sudo cp loopback /opt/cni/bin/
```

## 安装calico/kube-controllers

注意： 即使没有使用policy,也需要运行calico/kube-controllers

安装步骤：

- 下载 yaml文件  [Calico Kubernetes controllers manifest](https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/calico-kube-controllers.yaml).
- Modify `<ETCD_ENDPOINTS>` to point to your etcd cluster.
- Install it using `kubectl`.

```bash
$ kubectl create -f calico-kube-controllers.yaml
```

查看状态

```bash
$ kubectl get pods --namespace=kube-system
NAME                                     READY     STATUS    RESTARTS   AGE
calico-kube-controllers                  1/1       Running   0          1m
```

For more information on how to configure the controllers, see the [configuration guide](https://docs.projectcalico.org/v3.3/reference/kube-controllers/configuration).

## 权限控制 (RBAC)

在启用了RBAC的Kubernetes集群上安装Calico时，必须为某些Kubernetes API提供Calico访问权限。为此，必须在Kubernetes API中配置主题和角色，并且必须为Calico组件提供相应的令牌或证书，以将其标识为已配置的API用户。

配置Kubernetes RBAC的详细说明不在本文档的讨论范围之内, 参考文章 [upstream Kubernetes documentation](https://kubernetes.io/docs/admin/authorization/rbac/) 

以下YAML文件定义了Calico在使用etcd数据存储区时所需的必要API权限。

```bash
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/rbac.yaml
```

## Calico安装中默认不启用对应用层策略的支持

----

# Calico [张馆长博客](https://zhangguanzhang.github.io/2018/09/18/kubernetes-1-11-x-bin/)

Calico 是一款纯 Layer 3 的网络，其好处是它整合了各种云原生平台(Docker、Mesos 与 OpenStack 等)，且 Calico 不采用 vSwitch，而是在每个 Kubernetes 节点使用 vRouter 功能，并通过 Linux Kernel 既有的 L3 forwarding 功能，而当资料中心复杂度增加时，Calico 也可以利用 BGP route reflector 來达成。

> 想了解 Calico 与传统 overlay networks 的差异，可以阅读 [Difficulties with traditional overlay networks](https://www.projectcalico.org/learn/) 文章。

由于 Calico 提供了 Kubernetes resources YAML 文件来快速以容器方式部署网络插件至所有节点上，因此只需要在k8s-m1使用 kubeclt 执行下面指令來建立：

这边镜像因为是quay.io域名仓库会拉取很慢,所有节点可以提前拉取下,否则就等。镜像名根据输出来,可能我博客部分使用镜像版本更新了

```bash
$ grep -Po 'image:\s+\K\S+' addons/calico/v3.1/calico.yml 
quay.io/calico/typha:v0.7.4
quay.io/calico/node:v3.1.3
quay.io/calico/cni:v3.1.3
```



> 另外当节点超过 50 台，可以使用 Calico 的 Typha 模式来减少通过 Kubernetes datastore 造成 API Server 的负担。

包含上面三个镜像,拉取两个即可

```bash
curl -s https://zhangguanzhang.github.io/bash/pull.sh | bash -s -- quay.io/calico/node:v3.1.3
curl -s https://zhangguanzhang.github.io/bash/pull.sh | bash -s -- quay.io/calico/cni:v3.1.3
```



```bash
sed -ri "s#\{\{ interface \}\}#${interface}#" addons/calico/v3.1/calico.yml
kubectl apply -f addons/calico/v3.1
$ kubectl -n kube-system get pod --all-namespaces
NAMESPACE     NAME                              READY     STATUS              RESTARTS   AGE
kube-system   calico-node-2hdqf                 0/2       ContainerCreating   0          4m
kube-system   calico-node-456fh                 0/2       ContainerCreating   0          4m
kube-system   calico-node-jh6vd                 0/2       ContainerCreating   0          4m
kube-system   calico-node-sp6w9                 0/2       ContainerCreating   0          4m
kube-system   calicoctl-6dfc585667-24s9h        0/1       Pending             0          4m
kube-system   kube-proxy-46hr5                  1/1       Running             0          7m
kube-system   kube-proxy-l42sk                  1/1       Running             0          7m
kube-system   kube-proxy-p2nbf                  1/1       Running             0          7m
kube-system   kube-proxy-q6qn9                  1/1       Running             0          7m
```

calico正常是下面状态

```bash
$ kubectl get pod --all-namespaces
NAMESPACE     NAME                              READY     STATUS    RESTARTS   AGE
kube-system   calico-node-2hdqf                 2/2       Running   0          4m
kube-system   calico-node-456fh                 2/2       Running   2          4m
kube-system   calico-node-jh6vd                 2/2       Running   0          4m
kube-system   calico-node-sp6w9                 2/2       Running   0          4m
kube-system   calicoctl-6dfc585667-24s9h        1/1       Running   0          4m
kube-system   kube-proxy-46hr5                  1/1       Running   0          8m
kube-system   kube-proxy-l42sk                  1/1       Running   0          8m
kube-system   kube-proxy-p2nbf                  1/1       Running   0          8m
kube-system   kube-proxy-q6qn9                  1/1       Running   0          8m
```



部署后通过下面查看状态即使正常

```bash
kubectl -n kube-system get po -l k8s-app=calico-node
NAME                READY     STATUS    RESTARTS   AGE
calico-node-bv7r9   2/2       Running   4          5m
calico-node-cmh2w   2/2       Running   3          5m
calico-node-klzrz   2/2       Running   4          5m
calico-node-n4c9j   2/2       Running   4          5m
```



查找calicoctl的pod名字

```bash
kubectl -n kube-system get po -l k8s-app=calicoctl
NAME                         READY     STATUS    RESTARTS   AGE
calicoctl-6b5bf7cb74-d9gv8   1/1       Running   0          5m
```



通过 kubectl exec calicoctl pod 执行命令来检查功能是否正常

```bash
$ kubectl -n kube-system exec calicoctl-6b5bf7cb74-d9gv8 -- calicoctl get profiles -o wide
NAME              LABELS   
kns.default       map[]    
kns.kube-public   map[]    
kns.kube-system   map[]    

$ kubectl -n kube-system exec calicoctl-6b5bf7cb74-d9gv8 -- calicoctl get node -o wide
NAME     ASN         IPV4                 IPV6   
k8s-m1   (unknown)   192.168.88.111/24          
k8s-m2   (unknown)   192.168.88.112/24          
k8s-m3   (unknown)   192.168.88.113/24          
k8s-n1   (unknown)   10.244.3.1/24
```

完成后,通过检查节点是否不再是NotReady,以及 Pod 是否不再是Pending：

-----

## 附件：

rbca.yaml

```yaml
# Calico Version v3.3.1
# https://docs.projectcalico.org/v3.3/releases#v3.3.1

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-kube-controllers
rules:
  - apiGroups:
    - ""
    - extensions
    resources:
      - pods
      - namespaces
      - networkpolicies
      - nodes
      - serviceaccounts
    verbs:
      - watch
      - list
  - apiGroups:
    - networking.k8s.io
    resources:
      - networkpolicies
    verbs:
      - watch
      - list
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-kube-controllers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-kube-controllers
subjects:
- kind: ServiceAccount
  name: calico-kube-controllers
  namespace: kube-system

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: calico-node
rules:
  - apiGroups: [""]
    resources:
      - pods
      - nodes
      - namespaces
    verbs:
      - get
  - apiGroups: [""]
    resources:
      - nodes/status
    verbs:
      - patch

---

apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: calico-node
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: calico-node
subjects:
- kind: ServiceAccount
  name: calico-node
  namespace: kube-system
```



calico.yaml

```yaml
# Calico Version v3.3.1
# https://docs.projectcalico.org/v3.3/releases#v3.3.1
# This manifest includes the following component versions:
#   calico/node:v3.3.1
#   calico/cni:v3.3.1
#   calico/kube-controllers:v3.3.1

# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Configure this with the location of your etcd cluster.
  etcd_endpoints: "https://192.168.10.110:2379,https://192.168.10.111:2379,https://192.168.10.120:2379"

  # If you're using TLS enabled etcd uncomment the following.
  # You must also populate the Secret below with these files.
  etcd_ca: "/calico-secrets/etcd-ca"
  etcd_cert: "/calico-secrets/etcd-cert"
  etcd_key: "/calico-secrets/etcd-key"
  # Configure the Calico backend to use.
  calico_backend: "bird"

  # Configure the MTU to use
  veth_mtu: "1440"

  # The CNI network configuration to install on each node.  The special
  # values in this config will be automatically populated.
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.0",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "etcd_endpoints": "__ETCD_ENDPOINTS__",
          "etcd_key_file": "__ETCD_KEY_FILE__",
          "etcd_cert_file": "__ETCD_CERT_FILE__",
          "etcd_ca_cert_file": "__ETCD_CA_CERT_FILE__",
          "mtu": __CNI_MTU__,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "__KUBECONFIG_FILEPATH__"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        }
      ]
    }

---

# The following contains k8s Secrets for use with a TLS enabled etcd cluster.
# For information on populating Secrets, see http://kubernetes.io/docs/user-guide/secrets/
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following files with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # This self-hosted install expects three files with the following names.  The values
  # should be base64 encoded strings of the entire contents of each file.
  etcd-key: "LS0tLS1CRUdJTiBSU0Eg...VORCBSU0EgUFJJVkFURSBLRVktLS0tLQo="
  etcd-cert: "LS0tLS1CRUdJTiBDRVJ...0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
  etcd-ca: "LS0tLS1CRUdJTiBDRVJUS...0tRU5EIENFUlRJRklDQVRFLS0tLS0K"

---

# This manifest installs the calico/node container, as well
# as the Calico CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        k8s-app: calico-node
      annotations:
        # This, along with the CriticalAddonsOnly toleration below,
        # marks the pod as a critical add-on, ensuring it gets
        # priority scheduling and that its resources are reserved
        # if it ever gets evicted.
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
      hostNetwork: true
      tolerations:
        # Make sure calico-node gets scheduled on all nodes.
        - effect: NoSchedule
          operator: Exists
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - effect: NoExecute
          operator: Exists
      serviceAccountName: calico-node
      # Minimize downtime during a rolling upgrade or deletion; tell Kubernetes to do a "force
      # deletion": https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods.
      terminationGracePeriodSeconds: 0
      containers:
        # Runs calico/node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        - name: calico-node
          # image: quay.io/calico/node:v3.3.1
          image: marvinpan/calico-node:v3.3.1
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Set noderef for node controller.
            - name: CALICO_K8S_NODE_REF
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            # Choose the backend to use.
            - name: CALICO_NETWORKING_BACKEND
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: calico_backend
            # Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            # Auto-detect the BGP IP address.
            - name: IP
              value: "autodetect"
            # Enable IPIP
            - name: CALICO_IPV4POOL_IPIP
              value: "Never"
            # Set MTU for tunnel device used if ipip is enabled
            - name: FELIX_IPINIPMTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
            # The default IPv4 pool to create on startup if none exists. Pod IPs will be
            # chosen from this range. Changing this value after installation will have
            # no effect. This should fall within `--cluster-cidr`.
            - name: CALICO_IPV4POOL_CIDR
              value: "172.30.0.0/16"
            # Disable file logging so `kubectl logs` works.
            - name: CALICO_DISABLE_FILE_LOGGING
              value: "true"
            # Set Felix endpoint to host default action to ACCEPT.
            - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
              value: "ACCEPT"
            # Disable IPv6 on Kubernetes.
            - name: FELIX_IPV6SUPPORT
              value: "false"
            # Set Felix logging to "info"
            - name: FELIX_LOGSEVERITYSCREEN
              value: "info"
            - name: FELIX_HEALTHENABLED
              value: "true"
          securityContext:
            privileged: true
          resources:
            requests:
              cpu: 250m
          livenessProbe:
            httpGet:
              path: /liveness
              port: 9099
              host: localhost
            periodSeconds: 10
            initialDelaySeconds: 10
            failureThreshold: 6
          readinessProbe:
            exec:
              command:
              - /bin/calico-node
              - -bird-ready
              - -felix-ready
            periodSeconds: 10
          volumeMounts:
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
            - mountPath: /run/xtables.lock
              name: xtables-lock
              readOnly: false
            - mountPath: /var/run/calico
              name: var-run-calico
              readOnly: false
            - mountPath: /var/lib/calico
              name: var-lib-calico
              readOnly: false
            - mountPath: /calico-secrets
              name: etcd-certs
        # This container installs the Calico CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          # image: quay.io/calico/cni:v3.3.1
          image: marvinpan/calico-cni:v3.3.1
          command: ["/install-cni.sh"]
          env:
            # Name of the CNI config file to create.
            - name: CNI_CONF_NAME
              value: "10-calico.conflist"
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
            # CNI MTU Config variable
            - name: CNI_MTU
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: veth_mtu
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
            - mountPath: /calico-secrets
              name: etcd-certs
      volumes:
        # Used by calico/node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        - name: var-lib-calico
          hostPath:
            path: /var/lib/calico
        - name: xtables-lock
          hostPath:
            path: /run/xtables.lock
            type: FileOrCreate
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0400
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
  namespace: kube-system

---

# This manifest deploys the Calico Kubernetes controllers.
# See https://github.com/projectcalico/kube-controllers
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
  labels:
    k8s-app: calico-kube-controllers
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ''
spec:
  # The controllers can only have a single active instance.
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
      labels:
        k8s-app: calico-kube-controllers
    spec:
      nodeSelector:
        beta.kubernetes.io/os: linux
        node-role.kubernetes.io/master: ''
      # The controllers must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      tolerations:
        # Mark the pod as a critical add-on for rescheduling.
        - key: CriticalAddonsOnly
          operator: Exists
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      serviceAccountName: calico-kube-controllers
      containers:
        - name: calico-kube-controllers
          # image: quay.io/calico/kube-controllers:v3.3.1
          image: marvinpan/calico-kube-controllers:v3.3.1
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
            # Choose which controllers to run.
            - name: ENABLED_CONTROLLERS
              value: policy,namespace,serviceaccount,workloadendpoint,node
          volumeMounts:
            # Mount in the etcd TLS secrets.
            - mountPath: /calico-secrets
              name: etcd-certs
          readinessProbe:
            exec:
              command:
              - /usr/bin/check-status
              - -r
      volumes:
        # Mount in the etcd TLS secrets with mode 400.
        # See https://kubernetes.io/docs/concepts/configuration/secret/
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets
            defaultMode: 0400        
---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-kube-controllers
  namespace: kube-system
```

