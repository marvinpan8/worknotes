## docker cluster-store选项

![img](https://ws1.sinaimg.cn/large/9e792b8fgy1fmtzs7vftpj20rm0en0uy.jpg)

## etcd-calico(bgp)实现docker夸主机通信

![img](https://ws1.sinaimg.cn/large/9e792b8fgy1fmu1bi4bl6j20u70bsgmu.jpg)

### 配置calico网络

```
-  启动etcd
etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls=http://192.168.2.11:2379 --debug

- 启动docker
iptables -P FORWARD ACCEPT
systemctl stop docker
dockerd --cluster-store=etcd://192.168.2.11:2379


- 设置calico网络配置
mkdir -p /etc/calico/
cat > /etc/calico/calicoctl.cfg <<EOF
apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "http://192.168.2.11:2379"
EOF

- 启动calico
calicoctl node run
calicoctl node status

- n1创建calico驱动类型的global网卡
docker network rm cal_net1
docker network create --driver calico --ipam-driver calico-ipam cal_net1
docker network ls

- n1创建的网络会自动同步到n2

docker network create --driver calico --ipam-driver calico-ipam cal_net1

--driver calico           指定使用 calico 的 libnetwork CNM driver。
--ipam-driver calico-ipam 指定使用 calico 的 IPAM driver 管理 IP。
```

### 测试calico网络-2台node的容器互通

```
docker run --net cal_net1 --name b1 -itd busybox
docker exec -it b1 ip a

docker run --net cal_net1 --name b2 -itd busybox
docker exec -it b2 ip a


[root@n1 ~]# docker exec -it b1 ping 192.168.158.64
PING 192.168.158.64 (192.168.158.64): 56 data bytes
64 bytes from 192.168.158.64: seq=0 ttl=62 time=0.774 ms
```

### calico网络结构探究

![img](https://ws1.sinaimg.cn/large/9e792b8fgy1fmu18usmuaj212u0goqdd.jpg)

### 遇到的问题,docker global类型的网络不自动同步: 原因 etcd advertise-client-urls被错误指定为0.0.0.0了,需要指明具体的ip地址

```
- 正确姿势
etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls=http://192.168.2.11:2379 --debug

- 错误姿势
etcd --listen-client-urls http://0.0.0.0:2379 --advertise-client-urls=http://0.0.0.0:2379
```

![img](https://ws1.sinaimg.cn/large/9e792b8fgy1fmtzxhr48vj218m02amxb.jpg)

### calicoctl node run干了什么事

```
1,开启ip_forward
2,下载calico-node镜像,并启动

docker run --net=host --privileged \
    --name=calico-node -d \
    --restart=always -e NODENAME=n1.ma.com \
    -e CALICO_NETWORKING_BACKEND=bird \
    -e CALICO_LIBNETWORK_ENABLED=true \
    -e ETCD_ENDPOINTS=http://127.0.0.1:2379 \
    -v /var/log/calico:/var/log/calico \
    -v /var/run/calico:/var/run/calico \
    -v /lib/modules:/lib/modules \
    -v /run:/run -v /run/docker/plugins:/run/docker/plugins \
    -v /var/run/docker.sock:/var/run/docker.sock \
quay.io/calico/node:latest


3,写入etcd信息
```

### 定制 Calico 的 IP 池

```
- 创建自定义ipam网络
cat << EOF | calicoctl create -f -
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 17.2.0.0/16
EOF

docker network create --driver calico --ipam-driver calico-ipam --subnet=17.2.0.0/16 my_net

- 启动容器(固定ip地址方式)
docker run --net my_net --ip 17.2.3.11 -it busybox

- 查看网段
calicoctl node status
calicoctl get ipPool
```

### 定制 Calico 的 IP 池

```
- 创建自定义ipam网络
cat << EOF | calicoctl create -f -
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 17.2.0.0/16
EOF

docker network create --driver calico --ipam-driver calico-ipam --subnet=17.2.0.0/16 my_net

- 启动容器(固定ip地址方式)
docker run --net my_net --ip 17.2.3.11 -it busybox

- 查看网段
calicoctl node status
calicoctl get ipPool
```

cat << EOF | calicoctl create -f -

- apiVersion: v1
  kind: ipPool
  metadata:
  cidr: 17.2.0.0/16
  EOF

### calico默认网络策略

```
- calico默认网络策略
Calico 默认的 policy 规则是：容器只能与同一个 calico 网络中的容器通信。

- 查看calico默认策略
calicoctl get profile cal_net1 -o yaml
```

![img](https://ws1.sinaimg.cn/large/9e792b8fgy1fmu1xsmhfpj20km0i2ag4.jpg)

```
① 命名为 cal_net1，这就是 calico 网络 cal_net1 的 profile。
② 为 profile 添加一个 tag cal_net1。注意，这个 tag 虽然也叫 cal_net1，其实可以随便设置，这跟上面的 name: cal_net1 没有任何关系。此 tag 后面会用到。
③ egress  对从容器发出的数据包进行控制，当前没有任何限制。
④ ingress 对进入容器的数据包进行限制，当前设置是接收来自 tag cal_net1 的容器，根据第 ① 步设置我们知道，实际上就是只接收本网络的数据包，这也进一步解释了前面的实验结果。
```

### 定制calico默认网络策略

![img](https://ws1.sinaimg.cn/large/9e792b8fgy1fmu3egsfjgj20u70bsgn9.jpg)

```
Calico 能够让用户定义灵活的 policy 规则，精细化控制进出容器的流量，下面我们就来实践一个场景：
    创建一个新的 calico 网络 cal_web 并部署一个 httpd 容器 web1。
    定义 policy 允许 cal_net2 中的容器访问 web1 的 80 端口。


- 首先创建 cal_web。
docker network create --driver calico --ipam-driver calico-ipam cal_web
calicoctl get profile cal_web -o yaml

- 在 host1 中运行容器 web1，连接到 cal_web：
docker run --net cal_web --name web1 -d httpd
docker exec -it web1 ip a

- 创建net1
docker network create --driver calico --ipam-driver calico-ipam cal_net1
docker run --net cal_net1 --name b2 -itd busybox
docker exec -it b2 ip a
访问 web1的80
docker exec -it b2 wget x.x.x.x  #不通

- 创建 policy 文件 web.yml，内容为:
cat > web.yaml<<EOF
- apiVersion: v1
  kind: profile
  metadata:
    name: cal_web
  spec:
    ingress:
    - action: allow
      protocol: tcp
      source:
        tag: cal_net1
      destination:
        ports:
          - 80
EOF
calicoctl apply -f web.yaml
calicoctl get profile cal_web -o yaml

- 访问 web1的80
docker exec -it b2 wget x.x.x.x  #通了

- 在两个节点分别查看策略(一致)
calicoctl get profile cal_web -o yaml
```

![img](https://ws1.sinaimg.cn/large/9e792b8fgy1fmu3g6jzflj20ns04vwgo.jpg)

## calico最佳实战

#### calico数据转发流程

```
发数据
查路由
arp网关(网关有代理arp功能)
数据发到主机
根据主机路由表转发
```

#### 实现网络

参考: http://cizixs.com/2017/10/19/docker-calico-network

![img](https://images2017.cnblogs.com/blog/806469/201712/806469-20171228211912131-55855872.png)

#### calico数据库探究

使用etcd brower图形化查看etcd

```
docker run --name etcd-browser -p 0.0.0.0:8000:8000 --env ETCD_HOST=192.168.2.11 --env ETCD_PORT=2379 --env AUTH_PASS=doe -itd buddho/etcd-browser

注意端口
注意etcdip
[root@n1 ~]# etcdctl ls /calico
/calico/bgp
/calico/ipam
/calico/v1

[root@n1 ~]# etcdctl ls /calico/bgp/v1
/calico/bgp/v1/host
/calico/bgp/v1/global


[root@n1 ~]# etcdctl ls /calico/bgp/v1/global
/calico/bgp/v1/global/node_mesh
/calico/bgp/v1/global/as_num
/calico/bgp/v1/global/loglevel
/calico/bgp/v1/global/custom_filters


[root@n1 ~]# etcdctl get /calico/bgp/v1/global/as_num
64512

[root@n1 ~]# etcdctl ls /calico/bgp/v1/host
/calico/bgp/v1/host/n1.ma.com
/calico/bgp/v1/host/n2.ma.com


[root@n1 ~]# etcdctl ls /calico/bgp/v1/host/n1.ma.com
/calico/bgp/v1/host/n1.ma.com/ip_addr_v6
/calico/bgp/v1/host/n1.ma.com/network_v4
/calico/bgp/v1/host/n1.ma.com/ip_addr_v4

[root@n1 ~]# etcdctl get /calico/bgp/v1/host/n1.ma.com/ip_addr_v4
192.168.2.11

[root@n1 ~]# etcdctl get /calico/bgp/v1/host/n1.ma.com/network_v4
192.168.2.0/24
```

#### calico组件担任的角色

参考: https://docs.projectcalico.org/v1.6/reference/without-docker-networking/docker-container-lifecycle

http://www.youruncloud.com/blog/131.html

```
bird   实现了bgp协议
flex   调度执行者
confd  服务发现,通过flex修改配置

libnetwork-管理ip/接口


etcd
    ipam  节点地址记录
    bgp   bgp相关
    v2    policy
```

## 以前总结的calico零散知识

```
#!/usr/bin/env bash


docker stats

etcd --advertise-client-urls=http://0.0.0.0:2379 --listen-client-urls=http://0.0.0.0:2379 --enable-v2 --debug

vim /usr/lib/systemd/system/docker.service
# /etc/systemd/system/docker.service
--cluster-store=etcd://192.168.14.132:2379

systemctl daemon-reload
systemctl restart docker.service

[root@node1 ~]# ps -ef|grep docker
root       8122      1  0 Nov07 ?        00:01:01 /usr/bin/dockerd --cluster-store=etcd://192.168.14.132:2379

etcdctl ls
/docker


cd /usr/local/bin
wget https://github.com/projectcalico/calicoctl/releases/download/v1.6.1/calicoctl
chmod +x calicoctl

[root@node1 ~]# rpm -qa|grep etcd
etcd-3.2.5-1.el7.x86_64

mkdir /etc/calico
cat >> /etc/calico/calicoctl.cfg <<EOF
apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "http://192.168.14.132:2379"
EOF

calicoctl node run
calicoctl node run --ip=192.168.14.132

1,开启ip_forward
2,下载calico-node镜像,并启动
3,写入etcd信息

iptables -P FORWARD ACCEPT
etcdctl rm --recursive /calico
etcdctl rm --recursive /docker

# 可以看到bgp邻居已经建立起来了(14.132 14.133)
calicoctl node status

# 任意一台机器创建网络,另一台机器会同步过去的
docker network rm cal_net1
docker network create --driver calico --ipam-driver calico-ipam cal_net1

#+++++++++++++++++++++++++++
#  测试
#+++++++++++++++++++++++++++
# 14.132
docker container run --net cal_net1 --name bbox1 -tid busybox
docker exec bbox1 ip address
docker exec bbox1 route -n

# 14.133
docker container run --net cal_net1 --name bbox2 -tid busybox


docker exec bbox2 ip address
docker exec bbox2 ping  192.168.108.128


#+++++++++++++++++++++++++++
#  参考
#+++++++++++++++++++++++++++
https://mp.weixin.qq.com/s/VL72aVjU4KB3c2UTihl-DA
http://blog.csdn.net/felix_yujing/article/details/55213239


#+++++++++++++++++++++++++++
#  创建网段
#+++++++++++++++++++++++++++
calicoctl node status
calicoctl get ipPool
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 10.20.0.0/24
  spec:
    ipip:
      enabled: true
    nat-outgoing: true


另外一个测试
docker network create --driver calico --ipam-driver calico-ipam  --subnet 10.30.0.0/24 net1

docker network create --driver calico --ipam-driver calico-ipam  --subnet 10.30.0.0/24 net1
docker network create --driver calico --ipam-driver calico-ipam  --subnet 10.30.0.0/24 net2
docker network create --driver calico --ipam-driver calico-ipam  --subnet 10.30.0.0/24 net3

#node1
docker run --net net1 --name workload-A -tid busybox
docker run --net net2 --name workload-B -tid busybox
docker run --net net1 --name workload-C -tid busybox
#node2
docker run --net net3 --name workload-D -tid busybox
docker run --net net1 --name workload-E -tid busybox



#同一网络内的容器（即使不在同一节点主机上）可以使用容器名来访问
docker exec workload-A ping -c 4 workload-C.net1
docker exec workload-A ping -c 4 workload-E.net1
#不同网络内的容器需要使用容器ip来访问（使用容器名会报：bad address）
docker exec workload-A ping -c 2  `docker inspect --format "{{ .NetworkSettings.Networks.net2.IPAddress }}" workload-B`


#calico默认策略,同一网络内的容器是能相互通信的；不同网络内的容器相互是不通的。不同节点上属于同一网络的容器也是能相互通信的，这样就实现了容器的跨主机互连。



#+++++++++++++++++++++++++++
#  修改默认策略
#+++++++++++++++++++++++++++

cat << EOF | calicoctl apply -f -
- apiVersion: v1
  kind: profile
  metadata:
    name: cal_net12icmp
    labels:
      role: database
  spec:
    ingress:
    - action: allow
      protocol: icmp
      source:
        tag: net1
      destination:
        tag: net2
EOF





https://docs.projectcalico.org/v2.2/reference/public-cloud/aws
$ calicoctl apply -f - << EOF
apiVersion: v1
kind: ipPool
metadata:
  cidr: 192.168.0.0/16
spec:
  ipip:
    enabled: true
    mode: cross-subnet
  nat-outgoing: true
EOF

参考:
Docker网络解决方案-Calico部署记录
https://allgo.cc/2015/04/16/centos7%E7%BD%91%E5%8D%A1%E6%A1%A5%E6%8E%A5/
yum install bridge-utils
calico原理
http://www.cnblogs.com/kevingrace/p/6864804.html
#!/usr/bin/env bash
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx-udp-ingress-controller
  labels:
    k8s-app: nginx-udp-ingress-lb
  namespace: kube-system
spec:
  replicas: 1
  selector:
    k8s-app: nginx-udp-ingress-lb
  template:
    metadata:
      labels:
        k8s-app: nginx-udp-ingress-lb
        name: nginx-udp-ingress-lb
    spec:
      hostNetwork: true
      terminationGracePeriodSeconds: 60
      containers:
      #- image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.8
      - image: 192.168.1.103/k8s_public/nginx-ingress-controller:0.9.0-beta.5
        name: nginx-udp-ingress-lb
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        ports:
        - containerPort: 81
          hostPort: 81
        - containerPort: 443
          hostPort: 443
        - containerPort: 53
          hostPort: 53
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
        - --udp-services-configmap=$(POD_NAMESPACE)/nginx-udp-ingress-configmap

apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-configmap-example
data:
  53: "kube-system/kube-dns:53"
```