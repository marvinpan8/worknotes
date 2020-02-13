# 我的traefik部署

## 1. 安装keepalived

```bash
yum install -y gcc openssl-devel popt-devel ipvsadm
mkdir /k8s/keepalived && cd /k8s/keepalived
wget http://www.keepalived.org/software/keepalived-2.0.6.tar.gz
tar -zxvf keepalived-2.0.6.tar.gz && rm -f keepalived-2.0.6.tar.gz
cd keepalived-2.0.6 && mkdir keepalived-prefix
# --prefix：指定安装路径
./configure --prefix=$(pwd)/keepalived-prefix
# 安装
make && make install
#复制文件
cp keepalived/etc/init.d/keepalived /etc/init.d
cp keepalived/etc/sysconfig/keepalived /etc/sysconfig/
cd keepalived-prefix/ && mkdir /etc/keepalived
#复制文件
cp etc/keepalived/keepalived.conf /etc/keepalived/
```

配置文件`/etc/keepalived/keepalived.conf`文件内容如下：

```bash
! Configuration File for keepalived

global_defs {
   notification_email {
     pantj@investoday.com.cn
   }
   notification_email_from kaadmin@localhost
   smtp_server 192.168.10.115
   smtp_connect_timeout 30
    # 标识本节点的字条串，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
   router_id vmsrv-010-XXX
}

vrrp_instance VI_1 {
    state MASTER
    interface ens192
    virtual_router_id 88
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass investoday0755
    }
    virtual_ipaddress {
        192.168.10.88/24
    }
}
# virtual_server 192.168.10.88 80{
#     delay_loop 6
#     lb_algo wrr
#     lb_kind DR
#     nat_mask 255.255.255.0
#     persistence_timeout 900
#     protocol TCP

#     real_server 192.168.10.112 80{
#         weight 1
#         TCP_CHECK {
#             connect_timeout 10
#             nb_get_retry 3
#             delay_before_retry 3
#             connect_port 80
#         }
#     }
#     real_server 192.168.10.121 80{
#         weight 1
#         TCP_CHECK {
#             connect_timeout 10
#             nb_get_retry 3
#             delay_before_retry 3
#             connect_port 80
#         }
#     }
# }

# virtual_server 192.168.10.88 443{
#     delay_loop 6
#     lb_algo wrr
#     lb_kind DR
#     nat_mask 255.255.255.0
#     persistence_timeout 900
#     protocol TCP
    
#     real_server 192.168.10.112 443{
#         weight 1
#         TCP_CHECK {
#             connect_timeout 10
#             nb_get_retry 3
#             delay_before_retry 3
#             connect_port 80
#         }
#     }   
#     real_server 192.168.10.121 443{
#         weight 1
#         TCP_CHECK {
#             connect_timeout 10
#             nb_get_retry 3
#             delay_before_retry 3
#             connect_port 80
#         }
#     }   
# } 
```

- 查看是`ipvsadm --list --timeout`, 比如我的机器就会返回如下结果：

  Timeout (tcp tcpfin udp): 900 120 300
  这就表明我的tcp session的timeout时间是900秒。

- 就是virtual_server的`persistence_timeout `，意思就是在这个一定时间内会讲来自同一用户（根据ip来判断的）route到同一个real server。对于长连接类的应用，你肯定需要这么做。配置值最好跟lvs的配置的timeout一致,900。

- state BACKUP **主服务器必须设置为MASTER**

- interface ens192 **与服务器的网卡接口必须一致**

- virtual_router_id 51 **主、备服务器的id必须一致， 不能与其他keepalive集群相同**

- priority 90 **备份服务器的priority必须小于主服务器的priority**

- auth_pass investoday0755 **服务器的密码必须一致**

- 192.168.10.88/24 **虚拟IP地址，和服务器必须在同一网段并且没有使用**
### 验证keepalived是否安装成功
```bash
# 启动
systemctl enable keepalived && systemctl start keepalived
#查看日志文件
tail -f -n 500 /var/log/messages
#查看网卡情况
ip a show ens192
```

三台node都启动了keepalived后，观察eth0的IP，会在三台node的某一台上发现一个VIP是192.168.10.121。

```bash
$ ip addr show ens192
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:37:39:23 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.112/24 brd 192.168.10.255 scope global noprefixroute ens192
       valid_lft forever preferred_lft forever
    inet 192.168.10.88/24 scope global secondary ens192
       valid_lft forever preferred_lft forever
```

关掉拥有这个VIP主机上的keepalived，观察VIP是否漂移到了另外两台主机的其中之一上。

改造traefik-ds.yaml,**注意管理端口8080无法更改**

```bash
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
---
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: traefik-ingress-controller
  namespace: kube-system
  labels:
    k8s-app: traefik-ingress-lb
spec:
  template:
    metadata:
      labels:
        k8s-app: traefik-ingress-lb
        name: traefik-ingress-lb
    spec:
      serviceAccountName: traefik-ingress-controller
      terminationGracePeriodSeconds: 60
      restartPolicy: Always
      nodeSelector:
        edgenode: "true"
      containers:
      - image: traefik:1.7.7-alpine
        name: traefik-ingress-lb
        resources:
          limits:
            cpu: 300m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        - name: admin
          containerPort: 8080
        securityContext:
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        args:
        - --configFile=/k8s/traefik/conf/traefik.toml
        - --api
        - --kubernetes
        - --logLevel=INFO
        - --metrics
        - --metrics.prometheus
        - --web.metrics.prometheus
```

**注意**: [traefik官方的prometheus数据源grafana 展示json](https://github.com/containous/traefik/blob/master/contrib/grafana/traefik-kubernetes.json)

```bash
--kubernetes.ingressclass="traefik-external"
```

> Value of `kubernetes.io/ingress.class` annotation that identifies Ingress objects to be processed.
>
> If the parameter is non-empty, only Ingresses containing an annotation with the same value are processed. Otherwise, Ingresses missing the annotation, having an empty value, or with the value `traefik` are processed.
>
>  如果参数是非空的，则只处理包含具有相同值的注释的 Ingres 。 否则，将处理缺少注释、具有空值或带有值 `traefik` 的 Ingress 。例如：`traefik-internal`, `traefik-external`区别内外网
>
> [参考官方Github文档](<https://github.com/containous/traefik/blob/88ebac942ed8993d2ffa630efd7e422976c7e4f4/docs/content/providers/kubernetes-ingress.md>)

在ingress中增加`kubernetes.io/ingress.class`注释即可
```bash
metadata:
  annotations:
    kubernetes.io/ingress.class: traefik-external
```

我们使用了`nodeSelector`选择边缘节点来调度traefik-ingress-lb运行在它上面，所有你需要使用：

```bash
nodeSelector:
  node-role.kubernetes.io/edge: "edge"
tolerations:
  - key: "node-role.kubernetes.io/edge"
    value: "edge"
    effect: "NoSchedule"
```

查看DaemonSet的启动情况：

```Bash
$ kubectl -n kube-system get ds
NAME                 DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE-SELECTOR                              AGE
traefik-ingress-lb   3         3         3         3            3           ...edge=edge                              2h
```

现在就可以在外网通过192.168.10.88:80来访问到traefik ingress了。

## 使用域名访问Kubernetes中的服务

现在我们已经部署了以下服务：

- 三个边缘节点，使用Traefik作为Ingress controller
- 使用keepalived做的VIP（虚拟IP）172.20.0.119

这样在访问该IP的时候通过指定不同的`Host`来路由到kubernetes后端服务。这种方式访问每个Service时都需要指定`Host`，而同一个项目中的服务一般会在同一个Ingress中配置，使用`Path`来区分Service已经足够，这时候只要为VIP（172.20.0.119）来配置一个域名，所有的外部访问直接通过该域名来访问即可。

如下图所示：

![è¾¹ç¼èç¹æ¶æ](https://jimmysong.io/kubernetes-handbook/images/kubernetes-edge-node-architecture.png)

