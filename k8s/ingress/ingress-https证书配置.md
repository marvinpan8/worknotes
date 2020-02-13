 ### ca-config.json

```bash
mkdir -p /k8s/traefik/ssl && cd /k8s/traefik/ssl
cat << EOF | tee ca-config.json
{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "jrtz": {
         "expiry": "876000h",
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

### ca-csr.json

**CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。**

#### host ===SAN

- 无SAN(Subject Alternative Name)---CN: app.ma.com-即使地址栏的域名和CN一样也报错

- 无SAN(Subject Alternative Name)---CN: *.ma.com-即使地址栏的域名和CN一样也报错

- SAN含app.ma.com(Subject Alternative Name)---CN: *.ma.com-仅app.ma.com域名可访问

-  SAN含`*.ma.com`(Subject Alternative Name)---CN:* .ma.com-可用任意*.ma.com来访问

```bash
cat << EOF | tee ca-csr.json
{
    "CN": "Investoday",
    "hosts": [
    	"*.investoday.net",
    	"investoday.net"
  	],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Shenzhen",
            "ST": "Guangdong",
            "O": "Investoday",
            "OU": "R&D"
        }
    ]
}
EOF
```

#### 最后必须tls,生成tls.key,tls.crt

```bash
cfssl gencert -initca ./ca-csr.json | cfssljson -bare ca -
cfssl gencert -ca=./ca.pem -ca-key=./ca-key.pem -config=./ca-config.json -profile=jrtz ./ca-csr.json | cfssljson -bare tls
```

---

### ---pem转换crt,key文件（未使用到）

```bash
openssl x509 -outform der -in tls.pem -out tls.crt
openssl rsa -in tls-key.pem -out tls.key
```

### 创建一个secret，保存https证书。

```bash
cp tls.pem tls.crt
cp tls-key.pem tls.key
kubectl delete secret traefik-cert -n kube-system
kubectl create secret generic traefik-cert --from-file=./tls.crt --from-file=./tls.key -n kube-system
```

### 生成浏览器 client 证书 **ca.crt**

```bash
openssl x509 -outform der -in ca.pem -out ca.crt
```

---

### 创建一个configmap，保存traefix的配置。

这里的traefix中配置了把所有http请求全部rewrite为https的规则，并配置相应的证书位置：

`insecureSkipVerify = true`，该项配置指定了traefik在访问https后端的时候可以忽略TLS证书验证错误，从而使得https的后端，如kubernetes dashboard，可以像http后端一样直接通过traefik透出

```bash
$ vim traefik.toml
insecureSkipVerify = true
defaultEntryPoints = ["http","https"]
[entryPoints]
  [entryPoints.http]
  address = ":80"
    [entryPoints.http.redirect]
    entryPoint = "https"
  [entryPoints.https]
  address = ":443"
    [entryPoints.https.tls]
      [[entryPoints.https.tls.certificates]]
      certFile = "/k8s/traefik/ssl/tls.crt"
      keyFile = "/k8s/traefik/ssl/tls.key"

$ kubectl create configmap traefik-conf --from-file=/jrtz/k8s/ssl/traefik/traefik.toml -n kube-system
```



### 重新部署Traefix，这里主要是要关联创建的secret和configMap，并挂载相对应的主机目录。

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
      volumes:
      - name: ssl
        secret:
          secretName: traefik-cert
      - name: config
        configMap:
          name: traefik-conf
      containers:
      - image: traefik:1.7.7-alpine
        name: traefik-ingress-lb
        volumeMounts:
        - mountPath: "/k8s/traefik/ssl/"
          name: "ssl"
        - mountPath: "/k8s/traefik/conf/"
          name: "config"
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
          hostPort: 80
        - name: https
          containerPort: 443
          hostPort: 443
        - name: admin
          containerPort: 8080
          hostPort: 8080
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
```

重新部署

```bash
kubectl apply -f traefik-ds.yaml
```

---

## ingress额外增加tls

**先创建Secret，阿里云免费的是nginx证书**

```bash
$ kubectl create secret tls lyzt-ingress --cert=tls.crt --key=tls.key -n invest
#eg.
$ kubectl create secret tls dataapi-ing --cert=2640461_dataapi.lyzt.investoday.net.pem --key=2640461_dataapi.lyzt.investoday.net.key -n dataapi-prod
```

**可以不增加，增加了会替换traefik的证书, tls与rules平级**

```yaml
spec:
  rules:
  - host: lyzt.investoday.net
    http:
      paths:
      - path:
        backend:
          serviceName: test-ingress
          servicePort: 80
  tls:
  - hosts:
    - lyzt.investoday.net
    secretName: lyzt-ingress
  - hosts:
    - quote.investoday.net
    - xxxx.investoday.net
    secretName: quote-ingress
```

删除secret,ds, ing

```bash
cd /k8s/traefik
kubectl delete secret traefik-cert -n kube-system
kubectl delete ds traefik-ingress-controller -n kube-system
kubectl delete ing traefik-web-ui -n kube-system

kubectl apply -f traefik-ds.yaml
kubectl apply -f ui.yaml
```

