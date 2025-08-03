# k8s部署api6

### 创建 APISIX 配置文件：conf/config.yaml

```yaml
apisix:
  node_listen:
    - port: 9080
  enable_ipv6: false
  enable_http2: true
  enable_control: true
  control:
    ip: 0.0.0.0
    port: 9092
deployment:
  role: traditional
  role_control_plane:
    config_provider: etcd
  admin:
    admin_listen:
      port: 9180
    admin_key:
      - name: admin
        key: e8be69cadf2b1fc1239cd8d1ec216cad
        role: admin
      - name: viewer
        key: 77e080c4f7fe5dc12378980b8d5dd59d
        role: viewer
  etcd:
    host:
      - https://172.17.0.11:2379
      - https://172.17.0.12:2379
      - https://172.17.0.13:2379
    prefix: /apisix
    timeout: 30
    tls:
      cert: /etc/kubernetes/pki/etcd/peer.crt
      key: /etc/kubernetes/pki/etcd/peer.key
      ca: /etc/kubernetes/pki/etcd/ca.crt
      verify: true

plugin_attr:
  prometheus:
    export_addr:
      ip: 0.0.0.0
      port: 9091
```

| 参数               | 配置解释（根据个人情况修改）            |
| ------------------ | --------------------------------------- |
| apisix.node_listen | 设置 APISIX 的用户访问端口，默认是 9080 |
| apisix.control |	配置控制 API 所监听的地址和端口，地址默认监听所有 IPv4 和 IPv6 的网卡 |
| deployment.role |	配置了部署方式为 traditional，同时指定了配置中心是 etcd |
| deployment.admin |	这个部分配置了 admin API 所监听的端口，以及访问 admin API 所需的 KEY。具体 KEY 可以按照下面的命令生成：openssl rand -hex 16 |
| deployment.etcd.host | etcd集群地址 |
| deployment.etcd.user | 提供给apisix使用的用户名 |
| deployment.etcd.password | 用户名密码 |
| deployment.etcd.prefix | 前缀以/apisix开开头的key，具体可以更具etcd的认证配置更改 |

### 创建 APISIX Dashboard 配置文件：conf/dashboard.yaml

```yaml
conf:
  listen:
    #    host: "::"
    port: 9000
  allow_list:             # If we don't set any IP list, then any IP access is allowed by default.
    - 127.0.0.1           # The rules are checked in sequence until the first match is found.
    - ::1                 # In this example, access is allowed only for IPv4 network 127.0.0.1, and for IPv6 network ::1.
    - 0.0.0.0/0
    # It also support CIDR like 192.168.1.0/24 and 2001:0db8::/32
  etcd:
    endpoints:
      - "http://1.1.1.1:2379"
      - "http://1.1.1.2:2379"
      - "http://1.1.1.3:2379"
    username: "guyougao"
    password: "aaa123"
    mtls:
      key_file: ""          # Path of your self-signed client side key
      cert_file: ""         # Path of your self-signed client side cert
      ca_file: ""           # Path of your self-signed ca cert, the CA is used to sign callers' certificates
    prefix: /apisix     # apisix config's prefix in etcd, /apisix by default

  log:
    error_log:
      level: warn       # supports levels, lower to higher: debug, info, warn, error, panic, fatal
      file_path:
        logs/error.log  # supports relative path, absolute path, standard output
      # such as: logs/error.log, /tmp/logs/error.log, /dev/stdout, /dev/stderr
    access_log:
      file_path:
        logs/access.log
  security:
    content_security_policy: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; frame-src *"

authentication:
  secret:
    a9173abd74f794d52123651eba1de42d
  expire_time: 3600
  users:
    - username: user1   # username and password for login `manager api`
      password: "aaa123"
    - username: user2
      password: "aaa123"

plugins:
  - api-breaker
  - authz-casbin
  - authz-casdoor
  - authz-keycloak
  - aws-lambda
  - azure-functions
  - basic-auth
  # - batch-requests
  - clickhouse-logger
  - client-control
  - consumer-restriction
  - cors
  - csrf
  - datadog
  # - dubbo-proxy
  - echo
  - error-log-logger
  # - example-plugin
  - ext-plugin-post-req
  - ext-plugin-post-resp
  - ext-plugin-pre-req
  - fault-injection
  - file-logger
  - forward-auth
  - google-cloud-logging
  - grpc-transcode
  - grpc-web
  - gzip
  - hmac-auth
  - http-logger
  - ip-restriction
  - jwt-auth
  - kafka-logger
  - kafka-proxy
  - key-auth
  - ldap-auth
  - limit-conn
  - limit-count
  - limit-req
  - loggly
  # - log-rotate
  - mocking
  # - node-status
  - opa
  - openid-connect
  - opentelemetry
  - openwhisk
  - prometheus
  - proxy-cache
  - proxy-control
  - proxy-mirror
  - proxy-rewrite
  - public-api
  - real-ip
  - redirect
  - referer-restriction
  - request-id
  - request-validation
  - response-rewrite
  - rocketmq-logger
  - server-info
  - serverless-post-function
  - serverless-pre-function
  - skywalking
  - skywalking-logger
  - sls-logger
  - splunk-hec-logging
  - syslog
  - tcp-logger
  - traffic-split
  - ua-restriction
  - udp-logger
  - uri-blocker
  - wolf-rbac
  - zipkin
  - elasticsearch-logge
  - openfunction
  - tencent-cloud-cls
  - ai
  - cas-auth
```

​	这个配置也比较好理解，需要注意的是 Dashboard 是通过 etcd 对 APISIX 间接管理的，因此不需要直接配置 APISIX 的地址，同样上面是端口和 etcd 相关的配置，然后日志部分的配置改成了标准错误和标准输出，如果有需要可以改成具体的路径然后通过卷映射到容器内部。

​	下面 authentication 部分的 secret 是 JWT 认证的密钥，同样按照上面的方法生成一下（openssl rand -hex 16），下面 users 是页面的登录用户，这里初始化了两个用户。

​	最后 plugins 是支持的插件，这个比较多，所以这里就列了几个，其余的参考官方给出了样例来填写即可。

###  创建apisix  deploy 和 svc

```yaml
#常规部署需要修改的部分  用三个***表示
---
apiVersion: v1
kind: Service
metadata:
  #  ---------***---------
  #  设置服务名称
  name: cce-apisix-service
#  namespace: ctos
spec:
  selector:
    #    绑定的工作负载
    #      ---------***---------
    app: cce-apisix
    version: v1
  ports:
    - name: service-port1
      protocol: TCP
      port: 9180
      targetPort: 9180
    - name: service-port2
      protocol: TCP
      port: 9080
      targetPort: 9080
    - name: service-port3
      protocol: TCP
      port: 9091
      targetPort: 9091
    - name: service-port4
      protocol: TCP
      port: 9443
      targetPort: 9443
    - name: service-port5
      protocol: TCP
      port: 9092
      targetPort: 9092
  type: ClusterIP
#指定集群IP，暂时未使用
#  clusterIP: None
#  clusterIPs:
#    - None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  #      ---------***---------
  #  工作负载名称
  name: cce-apisix
  #  指定命名空间，可不指定，默认为default
#  namespace: ctos
spec:
  #      ---------***---------
  #  启动的实例个数
  replicas: 2
  selector:
    matchLabels:
      #      ---------***---------
      #  工作负载名称
      app: cce-apisix
      version: v1
  template:
    metadata:
      labels:
        #      ---------***---------
        #  工作负载名称
        app: cce-apisix
        version: v1
    spec:
      volumes:
        #      ---------***---------
        #            自定义卷积名称
        - name: vol-cce-apisix
          configMap:
            #      ---------***---------
            #            配置项名称
            name: cce-apisix
            defaultMode: 420
      containers:
        - name: container-apisix
          #      ---------***---------
          #          镜像地址： 域名/组织/容器名称:版本
          image: 镜像地址
          #          参数选择Always和IfNotPresent。Always参数开启后工作负载每次重启/升级均会重新拉取镜像，否则只会在节点上不存在同名同版本镜像时拉取镜像
          volumeMounts:
            - name: vol-cce-apisix
              readOnly: true
              mountPath: /usr/local/apisix/conf/config.yaml
              subPath: config.yaml
          imagePullPolicy: IfNotPresent
#          配置使用的密钥
#      imagePullSecrets:
#        - name: cce-secret
```

###  创建dashboard  deploy 和 svc

```yaml
#常规部署需要修改的部分  用三个***表示
---
apiVersion: v1
kind: Service
metadata:
  #  ---------***---------
  #  设置服务名称
  name: cce-apisix-dashboard-service
#  namespace: ctos
spec:
  selector:
    #    绑定的工作负载
    #      ---------***---------
    app: cce-apisix-dashboard
    version: v1
  ports:
    - name: service-port1
      protocol: TCP
      port: 9000
      targetPort: 9000
  type: ClusterIP
#指定集群IP，暂时未使用
#  clusterIP: None
#  clusterIPs:
#    - None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  #      ---------***---------
  #  工作负载名称
  name: cce-apisix-dashboard
  #  指定命名空间，可不指定，默认为default
#  namespace: ctos
spec:
  #      ---------***---------
  #  启动的实例个数
  replicas: 2
  selector:
    matchLabels:
      #      ---------***---------
      #  工作负载名称
      app: cce-apisix-dashboard
      version: v1
  template:
    metadata:
      labels:
        #      ---------***---------
        #  工作负载名称
        app: cce-apisix-dashboard
        version: v1
    spec:
      volumes:
        #      ---------***---------
        #            自定义卷积名称
        - name: vol-cce-apisix-dashboard
          configMap:
            #      ---------***---------
            #            配置项名称
            name: cce-apisix-dashboard
            defaultMode: 420
      containers:
        - name: container-apisix-dashboard
          #      ---------***---------
          #          镜像地址： 域名/组织/容器名称:版本
          image: 镜像地址
          #          参数选择Always和IfNotPresent。Always参数开启后工作负载每次重启/升级均会重新拉取镜像，否则只会在节点上不存在同名同版本镜像时拉取镜像
          volumeMounts:
            - name: vol-cce-apisix-dashboard
              readOnly: true
              mountPath: /usr/local/apisix-dashboard/conf/conf.yaml
              subPath: dashboard.yaml
          imagePullPolicy: IfNotPresent
#          配置使用的密钥
#      imagePullSecrets:
#        - name: cce-secret
```



