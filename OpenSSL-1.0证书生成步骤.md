# OpenSSL-1.0证书生成步骤

OpenSSL版本：**1.0.2k-fips**

OpenSSL官网：https://docs.openssl.org/1.0.2/

OpenSSL仓库：https://github.com/openssl/openssl     分支：OpenSSL_1_0_2k

参考PXC：https://docs.percona.com/percona-xtradb-cluster/8.0/encrypt-traffic.html?h=openssl#encrypt-replicationist-traffic

## 一、公共配置文件（不带SAN的IP）

- **进入证书目录**：cd /k8s/etcd/ssl
- **修改公司名**

> 特别注意：服务端证书、客户端证书、根证书， 三者的CN(Common Name)不可以一样，一样会有这个报错：error 18 at 0 depth lookup:self signed certificate。
>

```properties
cat << EOF | tee common.cnf
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
prompt             = no

[ req_distinguished_name ]
countryName                = CN
stateOrProvinceName        = GuangDong
localityName               = ShenZhen
0.organizationName         = changxing-tech
commonName                 = changxing
EOF
```


## 二、生成CA秘钥和证书

证书颁发机构用于验证证书上的签名。

### 1. 生成CA密钥文件：ca-key.pem

```properties
openssl genrsa 2048 > ca-key.pem
```

### 2. 生成CA证书文件（100年）：ca.pem

```properties
openssl req -new -x509 -nodes -days 36500 -key ca-key.pem -out ca.pem -config common.cnf
# 验证
openssl x509 -in ca.pem -text -noout
```



## 三、服务端证书生成

### 1. 生成服务器密钥文件：server-key.pem 和 server-req.pem

```properties
vim common.cnf
# 必须修改common.cnf的 commonName = changxing-server
openssl req -newkey rsa:2048 -days 36000 \
-nodes -keyout server-key.pem -out server-req.pem -config common.cnf
# ------验证--------------------------
openssl req -in server-req.pem -text -noout
```

### 2. 移除密码：server-key.pem

```properties
openssl rsa -in server-key.pem -out server-key.pem
```

### 3. 生成 extfile文件

- **SAN中修改服务端的可用IP**

```properties
cat << EOF | tee san.cnf
[ v3_ext ]
subjectAltName = @alt_names

[ alt_names ]
# DNS.1 = changxing-tech.com
# DNS.2 = *.changxing-tech.com
IP.1  = 127.0.0.1
IP.2  = 172.17.0.40
IP.3  = 172.17.0.41
IP.4  = 172.17.0.42
IP.5  = 172.17.0.77
EOF
```

### 4. 生成服务器证书文件：server-cert.pem

**根据 ca.pem + ca.pem + server-req.pem 三个文件生成 server-cert.pem**

```properties
openssl x509 -req -in server-req.pem -days 36000 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
  -out server-cert.pem \
  -extensions v3_ext \
  -extfile san.cnf
# ------查看验证--------------------------
openssl x509 -in server-cert.pem -text -noout 
# 重点检查 IP
openssl x509 -in server-cert.pem -text -noout | grep -A1 "Subject Alternative Name"
# 校验过期时间
openssl x509 -in server-cert.pem -text -noout | grep "After"
# 校验issuer和subject的CN不同
openssl x509 -in server-cert.pem -issuer -subject -noout

#  验证服务器证书是否由 CA 证书正确签名：
openssl verify -CAfile ca.pem server-cert.pem
# --------如果验证成功，输出-------------
server-cert.pem: OK

# --------删除请求文件-------------
rm -f server-req.pem
```



## 四、服务端peer证书生成

### 1. 生成服务器密钥文件：peer-key.pem 和 peer-req.pem

```properties
vim common.cnf
# 必须修改common.cnf的 commonName = changxing-peer
openssl req -newkey rsa:2048 -days 36000 \
-nodes -keyout peer-key.pem -out peer-req.pem -config common.cnf
# ------验证--------------------------
openssl req -in peer-req.pem -text -noout
```

### 2. 移除密码：peer-key.pem

```properties
openssl rsa -in peer-key.pem -out peer-key.pem
```

### 3. 生成 extfile文件

- **SAN中修改服务端的可用IP**

```properties
cat << EOF | tee san.cnf
[ v3_ext ]
subjectAltName = @alt_names

[ alt_names ]
# DNS.1 = changxing-tech.com
# DNS.2 = *.changxing-tech.com
IP.1  = 127.0.0.1
IP.2  = 172.17.0.40
IP.3  = 172.17.0.41
IP.4  = 172.17.0.42
EOF
```

### 4. 生成服务器peer证书文件：peer-cert.pem

**根据 ca.pem + ca.pem + peer-req.pem 三个文件生成 peer-cert.pem**

```properties
openssl x509 -req -in peer-req.pem -days 36000 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
  -out peer-cert.pem \
  -extensions v3_ext \
  -extfile san.cnf
# ------查看验证--------------------------
openssl x509 -in peer-cert.pem -text -noout 
# 重点检查 IP
openssl x509 -in peer-cert.pem -text -noout | grep -A1 "Subject Alternative Name"
# 校验过期时间
openssl x509 -in peer-cert.pem -text -noout | grep "After"
# 校验issuer和subject的CN不同
openssl x509 -in peer-cert.pem -issuer -subject -noout

#  验证服务器证书是否由 CA 证书正确签名：
openssl verify -CAfile ca.pem peer-cert.pem
# --------如果验证成功，输出-------------
peer-cert.pem: OK

# --------删除请求文件-------------
rm -f peer-req.pem
```




## 五、客户端秘钥和证书生成

### 1. 生成客户端密钥文件：client-key.pem和client-req.pem

- **必须修改common.cnf的 commonName，不能与以上重复**

```properties
vim common.cnf
# 必须修改common.cnf的 commonName = changxing-client
openssl req -newkey rsa:2048 -days 3600 \
-nodes -keyout client-key.pem -out client-req.pem -config common.cnf
# ------验证--------------------------
openssl req -in client-req.pem -text -noout
```

### 2. 移除密码：client-key.pem

```properties
openssl rsa -in client-key.pem -out client-key.pem
```

### 3. 生成 extfile文件

- **SAN中修改客户端的可用IP**

```properties
cat << EOF | tee san.cnf
[ v3_ext ]
subjectAltName = @alt_names

[ alt_names ]
# DNS.1 = changxing-tech.com
# DNS.2 = *.changxing-tech.com
IP.1  = 127.0.0.1
IP.2  = 172.17.0.40
IP.3  = 172.17.0.41
IP.4  = 172.17.0.42
IP.5  = 172.17.0.20
IP.6  = 172.17.0.21
IP.7  = 172.17.0.22
IP.8  = 172.17.0.23
EOF
```

### 4. 生成客户端证书文件：client-cert.pem

```properties
openssl x509 -req -in client-req.pem -days 36000 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
  -out client-cert.pem -extensions v3_ext -extfile san.cnf
# ------查看验证--------------------------
openssl x509 -in client-cert.pem -text -noout
# 重点检查 IP
openssl x509 -in client-cert.pem -text -noout | grep -A1 "Subject Alternative Name"
# 校验过期时间
openssl x509 -in client-cert.pem -text -noout | grep "After"
# 校验issuer和subject的CN不同
openssl x509 -in client-cert.pem -issuer -subject -noout
#  验证服务器证书是否由 CA 证书正确签名：
openssl verify -CAfile ca.pem client-cert.pem
# --------如果验证成功，输出-------------
client-cert.pem: OK
# --------删除请求文件-------------
rm -f client-req.pem
```

### 5. 将私钥转换为PKCS#8格式

```properties
# 使用OpenSSL转换私钥格式
openssl pkcs8 -topk8 -nocrypt -in client-key.pem -out client-pkcs8.key
```

### 6. 设置环境变量

```properties
OPENSSL_INCLUDE_DIR = D:\dev\OpenSSL-Win64\include
OPENSSL_ROOT_DIR = D:\dev\OpenSSL-Win64
```







---

## 六、 滚动升级证书（参考MySQL-PXC）

以下步骤说明当集群中有两个节点时如何升级用于保护复制流量的证书。

1. 重新启动第一个节点，并将[`socket.ssl_ca`](https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-provider-index.html#socketssl_ca)选项设置为单个文件中新旧证书的组合。

   例如，您可以将`old-ca.pem` 和的内容合并`new-ca.pem`为`upgrade-ca.pem`如下内容：

```properties
cat old-ca.pem > upgrade-ca.pem && \
cat new-ca.pem >> upgrade-ca.pem
```

   设置[`wsrep_provider_options`](https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-system-index.html#wsrep_provider_options)变量如下：

- 参考：https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-provider-index.html

```properties
wsrep_provider_options="socket.ssl=yes;socket.ssl_ca=/etc/mysql/certs/upgrade-ca.pem;socket.ssl_cert=/etc/mysql/certs/old-cert.pem;socket.ssl_key=/etc/mysql/certs/old-key.pem"
```

2. 重新启动第二个节点，并将[`socket.ssl_ca`](https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-provider-index.html#socketssl_ca)、[`socket.ssl_cert`](https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-provider-index.html#socketssl_cert)和[`socket.ssl_key`](https://docs.percona.com/percona-xtradb-cluster/8.0/wsrep-provider-index.html#socketssl_cert)选项设置为相应的新证书文件。

```properties
wsrep_provider_options="socket.ssl=yes;socket.ssl_ca=/etc/mysql/certs/new-ca.pem;socket.ssl_cert=/etc/mysql/certs/new-cert.pem;socket.ssl_key=/etc/mysql/certs/new-key.pem"
```

3. 按照上一步所述，使用新的证书文件重新启动第一个节点。

4. 您可以删除旧的证书文件。































