# 前言
- 创建jenkins的执行namspace所有权限
```bash
kubectl create rolebinding jenkins-admin-binding --clusterrole=admin --user=jenkins --namespace=XXX
```
- 创建通用invest的访问日志权限
```bash
kubectl create rolebinding invest-binding --clusterrole=invest --serviceaccount=kube-system:invest --namespace=XXX
```
- 创建个人用户的namspace所有权限
```bash
kubectl create rolebinding invest-binding --clusterrole=admin --serviceaccount=kube-system:wuzh --namespace=XXX
```

---
# 创建用户认证授权的kubeconfig文件

[参考宋净超文章](https://rootsongjc.gitbooks.io/kubernetes-handbook/content/guide/kubectl-user-authentication-authorization.html)

当我们安装好集群后，如果想要把 kubectl 命令交给用户使用，就不得不对用户的身份进行认证和对其权限做出限制。

下面以创建一个 zhangwc 用户并将其绑定到 dev 和 test 两个 namespace 为例说明。

## 创建 CA 证书和秘钥

**创建 zhangwc-csr.json 文件**

```json
cat << EOF | tee zhangwc-csr.json
{
  "CN": "zhangwc",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

**生成 CA 证书和私钥**

在 [创建 TLS 证书和秘钥](https://rootsongjc.gitbooks.io/kubernetes-handbook/content/practice/create-tls-and-secret-key.html) 一节中我们将生成的证书和秘钥放在了所有节点的 `/jrtz/k8s/ssl` 目录下，下面我们再在 master 节点上为 zhangwc 创建证书和秘钥，在 `/jrtz/k8s/ssl` 目录下执行以下命令：

执行该命令前请先确保该目录下已经包含如下文件：

ca-key.pem   ca.pem    ca-config.json     zhangwc-csr.json

```bash
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes zhangwc-csr.json | cfssljson -bare zhangwc
----------------------------------------------------------
2017/08/31 13:31:54 [INFO] generate received request
2017/08/31 13:31:54 [INFO] received CSR
2017/08/31 13:31:54 [INFO] generating key: rsa-2048
2017/08/31 13:31:55 [INFO] encoded CSR
2017/08/31 13:31:55 [INFO] signed certificate with serial number 43372632012323103879829229080989286813242051309
2017/08/31 13:31:55 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

这将生成如下文件：

```
zhangwc.csr  zhangwc-key.pem  zhangwc.pem
```

## 创建 kubeconfig 文件

```bash
# 设置集群参数
export KUBE_APISERVER="https://192.168.10.99:8443"
kubectl config set-cluster kubernetes \
--certificate-authority=/jrtz/k8s/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=zhangwc.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials zhangwc \
--client-certificate=/jrtz/k8s/ssl/zhangwc.pem \
--client-key=/jrtz/k8s/ssl/zhangwc-key.pem \
--embed-certs=true \
--kubeconfig=zhangwc.kubeconfig

# 设置上下文参数
kubectl config set-context kubernetes \
--cluster=kubernetes \
--user=zhangwc \
--namespace=monitoring \
--kubeconfig=zhangwc.kubeconfig

# 设置默认上下文
kubectl config use-context kubernetes --kubeconfig=zhangwc.kubeconfig
```

我们现在查看 kubectl 的 context：

```bash
kubectl config get-contexts
CURRENT   NAME              CLUSTER           AUTHINFO        NAMESPACE
*         kubernetes        kubernetes        admin
          default-context   default-cluster   default-admin
```

显示的用户仍然是 admin，这是因为 kubectl 使用了 `$HOME/.kube/config` 文件作为了默认的 context 配置，我们只需要将其用刚生成的 `zhangwc.kubeconfig` 文件替换即可。

```bash
cp -f ./zhangwc.kubeconfig /root/.kube/config
```

关于 kubeconfig 文件的更多信息请参考 [使用 kubeconfig 文件配置跨集群认证](https://rootsongjc.gitbooks.io/kubernetes-handbook/content/guide/authenticate-across-clusters-kubeconfig.html)。

## 1.  绑定多namespace完全访问权限

如果我们想限制 zhangwc 用户的行为，需要使用 RBAC创建角色绑定以将该用户的行为限制在某个或某几个 namespace 空间范围内，例如：

```bash
kubectl create rolebinding devuser-admin-binding --clusterrole=admin --user=zhangwc --namespace=dev
kubectl create rolebinding devuser-admin-binding --clusterrole=admin --user=zhangwc --namespace=test
# 例如：增加 jenkins 用户对 namespace=XXX 具有完全访问权限,这一步就够了
kubectl create rolebinding jenkins-admin-binding --clusterrole=admin --user=jenkins --namespace=XXX
```

这样 zhangwc 用户对 dev 和 test 两个 namespace 具有完全访问权限。



让我们来验证以下，现在我们在执行：

```bash
# 获取当前的 context
kubectl config get-contexts
CURRENT   NAME         CLUSTER      AUTHINFO   NAMESPACE
*         kubernetes   kubernetes   devuser    dev
*         kubernetes   kubernetes   devuser    test

# 无法访问 default namespace
kubectl get pods --namespace default
Error from server (Forbidden): User "devuser" cannot list pods in the namespace "default". (get pods)

# 默认访问的是 dev namespace，您也可以重新设置 context 让其默认访问 test namespace
kubectl get pods
No resources found.
```

现在 kubectl 命令默认使用的 context 就是 devuser 了，且该用户只能操作 dev 和 test 这两个 namespace，并拥有完全的访问权限。

可以使用我写的[create-user.sh脚本](https://github.com/rootsongjc/kubernetes-handbook/blob/master/tools/create-user/create-user.sh)来创建namespace和用户并授权，参考[说明](https://rootsongjc.gitbooks.io/kubernetes-handbook/content/tools/create-user/README.md)。

关于角色绑定的更多信息请参考 [RBAC——基于角色的访问控制](https://rootsongjc.gitbooks.io/kubernetes-handbook/content/guide/rbac.md)。

----

## 2.  绑定指定namespace的特定权限

模板:  注意不要ServiceAccount,   subjects:  kind:  ===>User

```yml
# apiVersion: v1
# kind: ServiceAccount
# metadata:
#   name: zhangwc
#   namespace: monitoring

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: role-zhangwc
  namespace: monitoring
rules:
- apiGroups: ["monitoring.coreos.com"]
  resources: ["prometheusrules"]
  verbs: ["get","list","create","update","patch"]

---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: role-bind-zhangwc
  namespace: monitoring
subjects:
- kind: User
  name: zhangwc
  namespace: monitoring
roleRef:
  kind: Role
  name: role-zhangwc
  apiGroup: rbac.authorization.k8s.io
```

---

# 创建只查看业务多命名空间日志的用户

```bash
cat << EOF | tee invest-role.yml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
 name: invest
rules:
- apiGroups: [""]
  resources: 
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  verbs: ["get","watch","list"] 
- apiGroups: [""]
  resources: 
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs: ["get","list","watch"]
- apiGroups:
  - apps
  resources:
  - controllerrevisions
  - daemonsets
  - deployments
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  - statefulsets/scale
  verbs: ["get","list","watch"]
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs: ["get","list","watch"]
EOF
```
创建 
```bash
kubectl apply -f invest-role.yml
```
创建SA
```bash
kubectl create sa invest -n kube-system
```
### 查看日志绑定命名空间

```bash
kubectl create rolebinding invest-binding --clusterrole=invest --serviceaccount=kube-system:invest --namespace=XXX
```

查看token 

```bash
kubectl -n kube-system describe secrets | sed -rn '/\sinvest-token-/,/^token/{/^token/s#\S+\s+##p}'
-------或者---------------------------------------------------
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep invest | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo $DASHBOARD_LOGIN_TOKEN
```

---

# 创建指定命名空间所有权限的用户token

创建SA
```bash
kubectl create sa wuzh -n kube-system
```
### 个人绑定命名空间

```bash
kubectl create rolebinding invest-binding --clusterrole=admin --serviceaccount=kube-system:wuzh --namespace=XXX
```

查看token 

```bash
kubectl -n kube-system describe secrets | sed -rn '/\swuzh-token-/,/^token/{/^token/s#\S+\s+##p}'
-------或者---------------------------------------------------
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep invest | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo $DASHBOARD_LOGIN_TOKEN
```

---

