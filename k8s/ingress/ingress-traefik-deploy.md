### 玩转K8S之如何访问业务应用（Traefik-ingress篇)

https://www.kubernetes.org.cn/4408.html

上篇懒得写了索性转载了一篇nginx-ingress的，本篇我们来看神器Traefik，我个人是比较看好和偏向与Traefik的，它轻便易用而且还有界面。

先介绍下什么是Traefik，Traefik是一个为了让部署微服务更加便捷而诞生的现代HTTP反向代理、负载均衡工具。 它支持多种后台 (Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, Zookeeper, BoltDB, Rest API, file…) 来自动化、动态的应用它的配置文件设置。

![architecture.png](https://s1.51cto.com/images/20180802/1533206391132683.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

 

为什么比较偏向域Traefik呢，下面来简单对比下。

ingress:

使用nginx作为前端负载均衡，通过ingress controller不断的和kubernetes api交互，实时获取后端service，pod等的变化，然后动态更新nginx配置，并刷新使配置生效，达到服务发现的目的。

traefik:

traefik本身设计的就能够实时跟kubernetes api交互，感知后端service，pod等的变化，自动更新配置并重载。

相对来说traefik更快速方便，同时支持更多的特性，使反向代理，负载均衡更直接更高效。

 

来看看如何部署，很简单先把源码clone下来。

```bash
[root@k8smaster ~]#  git clone https://github.com/containous/traefik.git
```

来看看目录下都有什么，顺便找到对应的K8S文件。

```bash
[root@k8smaster ~]# cd traefik/
[root@k8smaster traefik]# cd examples/
[root@k8smaster examples]# cd k8s
[root@k8smaster k8s]# ls
cheese-default-ingress.yaml  cheese-ingress.yaml   cheeses-ingress.yaml     traefik-ds.yaml    ui.yaml
cheese-deployments.yaml      cheese-services.yaml  traefik-deployment.yaml  traefik-rbac.yaml
[root@k8smaster k8s]# pwd
/root/traefik/examples/k8s
```

OK，到这一层就找到了所需的文件，一般呢只需要两个文件，第一个就是deployment和rbac。

 

原因呢很简单，在第一篇部署的时候我们就说了，由于在Kubernets1.6之后启用了RBAC鉴权机制，所以需配置ClusterRole以及ClusterRoleBinding来对api-server的进行相应权限的鉴权。

 

那rbac这个文件呢就是创建ClusterRole和ClusterRoleBinding的，至于deployment文件这里就不说了，相信看到本篇文章的童鞋已经对K8S有了基本认识。

 

开始创建rbac

```bash
[root@k8smaster k8s]# kubectl apply -f traefik-rbac.yaml 
clusterrole.rbac.authorization.k8s.io "traefik-ingress-controller" created
clusterrolebinding.rbac.authorization.k8s.io "traefik-ingress-controller" created
检查是否成功
[root@k8smaster k8s]# kubectl get clusterrolebinding
NAME                                                   AGE
cluster-admin                                          113d
flannel                                                113d
heapster                                               113d
kubeadm:kubelet-bootstrap                              113d
……….
traefik-ingress-controller                             3s
 
[root@k8smaster k8s]# kubectl get clusterrole
NAME                                                        AGE
admin                                                       113d
cluster-admin                                               113d
edit                                                        113d
flannel                                                     113d
```

可以看到clusterrole，clusterrolebinding都创建成功了，下面创建Traefik。

- 修改镜像版本号`image: traefik:1.7.7-alpine`

```bash
[root@k8smaster k8s]# kubectl apply -f traefik-deployment.yaml 
serviceaccount "traefik-ingress-controller" created
deployment.extensions "traefik-ingress-controller" created
service "traefik-ingress-service" created
 
检查是否成功
[root@k8smaster k8s]# kubectl get svc,deployment,pod -n kube-system
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                     AGE
heapster                        ClusterIP   10.106.236.144   <none>        80/TCP                                      113d
kube-dns                        ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP                               113d
kubernetes-dashboard-external   NodePort    10.108.106.113   <none>        9090:30090/TCP                              113d
traefik-ingress-service         NodePort    10.98.76.58      <none>        80:30883/TCP ,8080:30731/TCP   17s
 
NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
heapster                     1         1         1            1           113d
kube-dns                     1         1         1            1           113d
kubernetes-dashboard         1         1         1            1           113d
traefik-ingress-controller   1         1         1            0           18s
 
NAME                                         READY     STATUS    RESTARTS   AGE
etcd-k8smaster                               1/1       Running   6          113d
heapster-6595c54cb9-f7gvz                    1/1       Running   4          113d
kube-apiserver-k8smaster                     1/1       Running   6          113d
……….
traefik-ingress-controller-bf6486db6-jzd8w   1/1       Running   0          17s
```

可以看到service和pod都起来了。

 

刚才前面也说到了有个非常简洁漂亮的界面，非常适合运维统计管理，下面来看看。

```bash
[root@k8smaster k8s]# cat ui.yaml 
---
apiVersion: v1
kind: Service
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  selector:
    k8s-app: traefik-ingress-lb
  ports:
  - name: web
    port: 80
    targetPort: 8080
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: traefik-web-ui
  namespace: kube-system
spec:
  rules:
  - host: traefik-ui.minikube
    http:
      paths:
      - path: /
        backend:
          serviceName: traefik-web-ui
          servicePort: web
 
[root@k8smaster k8s]# kubectl apply -f ui.yaml
service "traefik-web-ui" created
ingress.extensions "traefik-web-ui" created
[root@k8smaster k8s]# kubectl describe ing traefik-web-ui -n kube-system
Name:             traefik-web-ui
Namespace:        kube-system
Address:          
Default backend:  default-http-backend:80 (<none>)
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  traefik-ui.minikube  
                       /   traefik-web-ui:web (10.0.100.203:8080,10.0.100.204:8080)
```

刚才发布了一个traefix-web-ui的ingress，接下来我们就可以通过域名了访问了，玩过K8S的相信都能看懂刚才ui-ingress那个yml文件里面有一个域名，名为traefik-ui.minikube，后端traefix-web-ui的service，可以看到关联到了pod地址10.0.100.203:8080和10.0.100.204:8080。

 

下面我们修改本机hosts文件，使我们可以通过traefik.investoday.io域名来访问traefix-ui

![博客01.png](https://s1.51cto.com/images/20180802/1533206440716099.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=)

 

好了本文到此结束，本篇文章只是初步实现了Traefix的http访问代理，怎么让traefix实现https代理以及怎么对traefix进行更多的配置，将在后续的博文中来讨论。

本文参考资料：

<http://traefik.cn/>

<http://blog.51cto.com/goome/2151353>

---

## 健康检查

关于健康检查，测试可以使用 kubernetes 的 Liveness Probe 实现，如果 Liveness Probe检查失败，则 traefik 会自动移除该 [pod](https://www.kubernetes.org.cn/tags/pod)，以下是一个 示例

**test 的 deployment，健康检查方式是 cat /tmp/health，容器启动 2 分钟后会删掉这个文件，模拟健康检查失败**

```yaml
apiVersion: v1
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: test
  namespace: default
  labels:
    test: alpine
spec:
  replicas: 3
  selector:
    matchLabels:
      test: alpine
  template:
    metadata:
      labels:
        test: alpine
        name: test
    spec:
      containers:
      - image: mritd/alpine:3.4
        name: alpine
        resources:
          limits:
            cpu: 200m
            memory: 30Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
        args:
        command:
        - "bash"
        - "-c"
        - "echo ok > /tmp/health;sleep 120;rm -f /tmp/health"
        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/health
          initialDelaySeconds: 20
```

**test 的 service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test 
  labels:
    name: test
spec:
  ports:
  - port: 8123
    targetPort: 80
  selector:
    name: test
```

**test 的 Ingress**

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: test.com
    http:
      paths:
      - path: /
        backend:
          serviceName: test
          servicePort: 8123
```

**全部创建好以后，进入 traefik ui 界面，可以观察到每隔 2 分钟健康检查失败后，kubernetes 重建 pod，同时 traefik 会从后端列表中移除这个 pod**