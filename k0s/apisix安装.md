---

# ■■■ 安装apisix

## etcd 若是单节点需要创建更换crt证书

## 安装及配置CFSSL

```properties
#在任一目录下下载
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
#修改为可执行权限
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64
#移动到bin目录
mv cfssl_linux-amd64 /usr/local/bin/cfssl && \
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson && \
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
#验证
$ cfssl version
```
##### 创建 ETCD 证书
```properties
# 创建目录
mkdir -p /k0s/etcd && cd /k0s/etcd
cat << EOF | tee ca-config.json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "server": {
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

##### 创建 ETCD CA 配置文件

*   “CN”：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
*   “O”：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；
*   这两个参数在后面的kubernetes启用RBAC模式中很重要，因为需要设置kubelet、admin等角色权限，那么在配置证书的时候就必须配置对了，具体后面在部署kubernetes的时候会进行讲解。

```properties
cat << EOF | tee ca-csr.json
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "XS",
            "ST": "Shenzhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

#### 创建 ETCD Server 证书

- 注意修改ETCD监听的IP

```properties
cat << EOF | tee etcd-csr.json
{
    "CN": "etcd",
    "hosts": [
      "127.0.0.1",
      "172.17.0.20",
      "172.17.0.21",
      "172.17.0.22",
      "172.17.0.23"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "XS",
            "ST": "Shenzhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

#### 拷贝【20】节点原先生成的 ca.crt   ca.key

- 在目录下  /var/lib/k0s/pki/etcd

#### 生成 ETCD CA 证书和私钥

生成三个文件\*.csr是个中间证书请求文件，我们最终要的是

- ca-key.pem和ca.pem,
- etcd-key.pem 和 etcd.pem

```properties
cfssl gencert \
      -initca ca-csr.json | cfssljson -bare ca -
cfssl gencert \
      -ca=ca.crt \
      -ca-key=ca.key \
      -config=ca-config.json -profile=server etcd-csr.json | cfssljson -bare etcd
```

## 替换原先的 peer.crt peer.key

## 重启k0scontroller

```properties
systemctl restart k0scontroller
systemctl status k0scontroller
journalctl -f -u k0scontroller
```



### 【20】主节点拷贝etcd的ca cert key三个文件

```properties
ps -ef |grep etcd
---------------------------
--etcd-cafile=/var/lib/k0s/pki/etcd/ca.crt 
--cert-file=/var/lib/k0s/pki/etcd/server.crt 
--key-file=/var/lib/k0s/pki/etcd/server.key
# 拷贝文件
mkdir /k0s/etcd
cp /var/lib/k0s/pki/etcd/ca.crt /k0s/etcd
cp /var/lib/k0s/pki/server.crt  /k0s/etcd
cp /var/lib/k0s/pki/server.key  /k0s/etcd
```

## 【40】节点部署

```properties
kubectl create ns apisix
kubectl delete cm etcd-ca -n apisix
kubectl delete cm etcd-cert -n apisix
kubectl delete cm etcd-key -n apisix
kubectl create cm etcd-ca --from-file=/k0s/etcd/ca.crt -n apisix
kubectl create cm etcd-cert --from-file=/k0s/etcd/server.crt -n apisix
kubectl create cm etcd-key --from-file=/k0s/etcd/server.key -n apisix

kubectl delete cm apisix-config -n apisix
kubectl delete cm dashboard-config -n apisix
kubectl create cm apisix-config --from-file=/k0s/apisix/config/apisix-config.yaml -n apisix
kubectl create cm dashboard-config --from-file=/k0s/apisix/config/dashboard-config.yaml -n apisix

kubectl delete -f /k0s/apisix/apisix-deploy.yaml
kubectl delete -f /k0s/apisix/dashboard-depoy.yaml
kubectl apply -f /k0s/apisix/apisix-deploy.yaml
kubectl apply -f /k0s/apisix/dashboard-depoy.yaml
```

## apisix-dashboard

```properties
manager-api -p /usr/local/apisix/dashboard
systemctl daemon-reload && systemctl enable apisix-dashboard
systemctl restart apisix-dashboard

systemctl stop apisix-dashboard
systemctl status apisix-dashboard
journalctl -f -u apisix-dashboard
###---------------
tail -f -n 500 /usr/local/apisix/dashboard/logs/error.log
```

##  手动删除污点，使控制节点可以部署

```properties

# 查看污点：
kubectl describe node nodename |grep Taints
# 删除污点：
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

## deploy安装ETCD

```properties
# deploy 设置用户:    securityContext:  =>  fsGroup:  1001
kubectl apply -f etcd.yaml
rm -rf /data/etcd/*

chown 1001:root /data/etcd
chmod -R 755 /data/etcd
```

## apisix容器内安装ps、ss命令检查端口

```properties
yum install procps-ng iproute 
```




# 2. APISIX通过hostNetwork安装

```properties
kubectl get nodes --show-labels

chmod 644 /var/lib/k0s/pki/etcd/ca.crt
chmod 644 /var/lib/k0s/pki/apiserver-etcd-client.crt
chmod 644 /var/lib/k0s/pki/apiserver-etcd-client.key
```



## k0s中的ETCD集群校验

```properties
ETCDCTL_API=3 /usr/local/bin/etcdctl -w table --cacert=/var/lib/k0s/pki/etcd/ca.crt --cert=/var/lib/k0s/pki/apiserver-etcd-client.crt --key=/var/lib/k0s/pki/apiserver-etcd-client.key --endpoints="https://127.0.0.1:2379" member list

ETCDCTL_API=3 /usr/local/bin/etcdctl -w table --cacert=/var/lib/k0s/pki/etcd/ca.crt --cert=/var/lib/k0s/pki/apiserver-etcd-client.crt --key=/var/lib/k0s/pki/apiserver-etcd-client.key --endpoints="https://127.0.0.1:2379" endpoint health

ETCDCTL_API=3 /usr/local/bin/etcdctl -w table --cacert=/var/lib/k0s/pki/etcd/ca.crt --cert=/var/lib/k0s/pki/apiserver-etcd-client.crt --key=/var/lib/k0s/pki/apiserver-etcd-client.key --endpoints="https://127.0.0.1:2379" endpoint status

ETCDCTL_API=3 /usr/local/bin/etcdctl --cacert=/var/lib/k0s/pki/etcd/ca.crt --cert=/var/lib/k0s/pki/apiserver-etcd-client.crt --key=/var/lib/k0s/pki/apiserver-etcd-client.key --endpoints="https://127.0.0.1:2379" get //apisix/routes --prefix --keys-only
# 删除操作（慎用）
ETCDCTL_API=3 /usr/local/bin/etcdctl --cacert=/var/lib/k0s/pki/etcd/ca.crt --cert=/var/lib/k0s/pki/apiserver-etcd-client.crt --key=/var/lib/k0s/pki/apiserver-etcd-client.key --endpoints="https://127.0.0.1:2379" del /apisix000 --prefix
```