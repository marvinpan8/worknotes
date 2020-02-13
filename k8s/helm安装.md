### Helm Tiller Server

[Helm](https://github.com/kubernetes/helm)是Kubernetes Chart的管理工具,Kubernetes Chart是一套预先组态的Kubernetes资源套件。其中`Tiller Server`主要负责接收来至Client的指令,并通过kube-apiserver与Kubernetes集群做沟通,根据Chart定义的内容,来产生与管理各种对应API物件的Kubernetes部署文件(又称为`Release`)。

在所有`node`机器安裝 socat(用于端口转发)：

```bash
yum install -y socat
```

首先在`k8s-m1`安装Helm tool：

```bash
$ wget -qO- https://kubernetes-helm.storage.googleapis.com/helm-v2.13.1-linux-amd64.tar.gz | tar -zx
$ sudo mv linux-amd64/helm /usr/local/bin/
```

接着初始化 Helm(这边会安装 Tiller Server)：

```bash
$ kubectl -n kube-system create sa tiller
$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
# 替换国内阿里源
$ helm init --client-only --stable-repo-url https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts/
$ helm repo add incubator https://aliacs-app-catalog.oss-cn-hangzhou.aliyuncs.com/charts-incubator/
$ helm repo update

# 创建服务端
helm init --service-account tiller --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.9.1  --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
 
# 创建TLS认证服务端，参考地址：https://github.com/gjmzj/kubeasz/blob/master/docs/guide/helm.md
$ helm init --service-account tiller --upgrade -i marvinpan/gcr.io.kubernetes-helm.tiller:v2.13.1 --tiller-namespace kube-system --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

...
Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
Happy Helming!
```



这边默认helm的部署的镜像是`gcr.io/kubernetes-helm/tiller:v2.13.1`,如果拉取不了可以使用命令修改成国内能拉取到的我的镜像 `marvinpan/gcr.io.kubernetes-helm.tiller:v2.13.1`

```bash
kubectl -n kube-system patch deploy  tiller-deploy -p '{"spec":{"template":{"spec":{"containers":[{"name":"tiller","image":"marvinpan/gcr.io.kubernetes-helm.tiller:v2.13.1"}]}}}}'
```

修改调度策略，专用monitoring节点上

```bash
kubectl edit deploy tiller-deploy -n kube-system
# 增加 nodeSelector 和 tolerations
nodeSelector:
  node-role.kubernetes.io/monitoring: "monitoring"
tolerations:
  - key: "node-role.kubernetes.io/monitoring"
    operator: "Equal"
    value: "monitoring"
    effect: "NoSchedule"
```

查看tiller的pod

```bash
$ kubectl -n kube-system get po -l app=helm
NAME                             READY     STATUS    RESTARTS   AGE
tiller-deploy-5f789bd9f7-tzss6   1/1       Running   0          29s

$ helm version
Client: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.1", GitCommit:"618447cbf203d147601b4b9bd7f8c37a5d39fbb4", GitTreeState:"clean"}
```



**测试Helm 功能**
这边部署简单Jenkins 来进行功能测试：

```bash
$ helm install --name demo --set Persistence.Enabled=false stable/jenkins
```



查看状态

```bash
$ kubectl get po,svc  -l app=demo-jenkins
NAME                           READY     STATUS    RESTARTS   AGE
demo-jenkins-7bf4bfcff-q74nt   1/1       Running   0          2m

NAME                 TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
demo-jenkins         LoadBalancer   10.103.15.129    <pending>     8080:31161/TCP   2m
demo-jenkins-agent   ClusterIP      10.103.160.88   <none>        50000/TCP        2m
```



取得 admin 账号的密码

```bash
$ printf $(kubectl get secret --namespace default demo-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
r6y9FMuF2u
```



可以上面状态看到nodeport的端口为`31161`
完成后,就可以通过浏览器访问Jenkins Web [http://192.168.88.110:31161](http://192.168.88.110:31161/)。

[![cmd-markdown-logo](https://kairen.github.io/images/kube/helm-jenkins-v1.10.png)](https://kairen.github.io/images/kube/helm-jenkins-v1.10.png)
测试完成后,即可删除：

```bash
$ helm ls
NAME    REVISION    UPDATED                     STATUS      CHART             NAMESPACE
demo    1           Tue Apr 10 07:29:51 2018    DEPLOYED    jenkins-0.14.4    default

$ helm delete demo --purge
release "demo" deleted
```



更多Helm Apps可以到[Kubeapps Hub](https://hub.kubeapps.com/)寻找。