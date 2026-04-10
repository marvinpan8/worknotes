# k0s部署 calico 配置交换机

**calico 版本：3.31.4**    https://github.com/projectcalico/calico

**Metallb 版本：0.15.3**

## 一、calico 配置

### 下载 calicoctl

```properties
# 下载 二进制
curl -L https://github.com/projectcalico/calico/releases/download/v3.31.4/calicoctl-linux-amd64
chmod +x calicoctl-linux-amd64
sudo cp calicoctl-linux-amd64 /usr/local/bin/kubectl-calico
sudo cp calicoctl-linux-amd64 /usr/local/bin/calicoctl

# 查看 Calico 节点
kubectl calico get nodes
# 查看 IP 池
kubectl calico get ippools
# 查看 BGP 状态
sudo kubectl calico node status
# 查看 BGP 配置（如果有）
calicoctl get bgpconfigurations
# 查看 BGP Peer（如果有）
calicoctl get bgppeers
```

bgpconfig:  https://docs.tigera.io/calico/latest/reference/resources/bgpconfig

bgppeer: https://docs.tigera.io/calico/latest/reference/resources/bgppeer

```properties
apiVersion: crd.projectcalico.org/v1
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: false
  asNumber: 65002
  serviceLoadBalancerIPs:
    - cidr: 10.10.10.0/24
---
apiVersion: crd.projectcalico.org/v1
kind: BGPPeer
metadata:
  name: h3c-switch
spec:
  peerIP: 10.10.10.1
  asNumber: 65001
```

- **BGPConfiguration： **
  - **nodeToNodeMeshEnabled: false** 不启用节点间Mesh，由交换机接管
  - **asNumber: k8s AS** 
  - **serviceClusterIPs** 不配置默认为空，不需要也不应该上报给交换机
- **BGPPeer： **
  - **peerIP：交换机IP**
  - **asNumber: 交换机AS**
  - **nodeSelector** 不配置默认为空，否则导致不上报的节点的pod 无法访问

###  检查验证

```properties
k get po -o wide|grep calico
kubectl logs -n kube-system calico-node-kksvh | grep -i "bgp\|bird\|peer"
kubectl -n kube-system exec calico-node-kksvh -- calico-node -show-status
```
### 添加网络策略

```properties
apiVersion: crd.projectcalico.org/v1
kind: HostEndpoint
metadata:
  name: app102-endpoint
  labels:
    node: app102
    role: docker-host
spec:
  interfaceName: bond0
  node: app102
  expectedIPs:
    - 10.10.10.102
---
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: allow-docker-5000-access
spec:
  order: 5
  selector: node == "app102"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        nets:
          - 10.10.10.0/24
      destination:
        ports:
          - 5000
  egress:
    - action: Allow
      protocol: TCP
      destination:
        ports:
          - 5000
---
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: allow-node-to-node
spec:
  order: 5
  selector: all()
  applyOnForward: true
  ingress:
    - action: Allow
      protocol: TCP
      source:
        nets:
          - 10.10.10.0/24
      destination:
        nets:
          - 10.10.10.0/24
        ports:
          - 10250  # kubelet API
    - action: Allow
      protocol: ICMP
      source:
        nets:
          - 10.10.10.0/24
      destination:
        nets:
          - 10.10.10.0/24
```

## 二、Metallb controller安装

- 官网：https://metallb.io/
- **version: 0.15.3**  GitHub: https://github.com/metallb/metallb.git

- **config/manifests/metallb-native.yaml  删除所有speaker 配置，仅保留controller **

```properties
kubeclt apply metallb-native-controller.yaml
```

### 配置 IPAddressPool   BGPAdvertisement

```properties
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: bgp-pool
  namespace: metallb-system
spec:
  addresses:
  - 10.10.10.128/25
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: BGPAdvertisement
metadata:
  name: bgp-adv
  namespace: metallb-system
spec:
  ipAddressPools:
  - bgp-pool
  aggregationLength: 32
```

## 三、综合验证

```properties
# 查看所有 LoadBalancer 服务
kubectl get svc -A | grep LoadBalancer
# LoadBalancer 服务正常
curl http://10.10.10.128:8000/get?show_env=true

sudo iptables -L > iptables-101.log
#-----calico 自动生成 DROP 节点IP  ------------------------------------------
Chain cali-cidr-block (1 references)
target     prot opt source               destination         
DROP       all  --  anywhere             app102/25            /* cali:rQCL9AF5lrNpX_HO */
#--------------------
sudo ipvsadm -Ln | grep 30233
# 在源主机上 请求
curl http://10.10.10.101:30233/get?show_env=true
# 在目标主机 101 上 抓包
sudo tcpdump -i any host 10.10.10.101 and port 30233 -n

curl http://10.10.10.128:8000/get?show_env=true
curl http://10.10.10.103:30233/get?show_env=true
curl http://10.74.183.127:8000/get?show_env=true
curl http://httpbin:8000/get?show_env=true
curl http://10.244.109.198:8000/get?show_env=true
```



























