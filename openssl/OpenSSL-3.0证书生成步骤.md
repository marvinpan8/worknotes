# OpenSSL-3.0证书生成步骤

- Ubantu 24.04
- OpenSSL：**3.0.13-0ubuntu3.7（ubantu 自带版本）**

- 参考PXC：https://docs.percona.com/percona-xtradb-cluster/8.4/encrypt-traffic.html#generate-keys-and-certificates-manually

```properties
sudo mkdir -p /etc/mysql/certs
sudo chown -R ekemp:ekemp /etc/mysql/certs
```

## 一、公共配置文件（不带SAN的IP）

- **进入证书目录**：cd /k8s/etcd/ssl
- **修改 commonName必须不一致，对于服务器证书，通常是域名（如 `example.com`）**。现代浏览器和操作系统（以及你使用的 OpenSSL 1.1.1+ / 3.x）**更优先使用 SAN（Subject Alternative Name，主题备用名称）扩展**。`CN` 逐渐退居二线，但仍然是标识证书**所有者**的核心字段。**最佳实践：**将主要域名放在 CN 中。将所有需要访问的域名（包括主域名和子域名）都列在 [ **alt_names** ] 段落中。
- **distinguished_name = req_distinguished_name**  指的是有 [ req_distinguished_name ] 段落
- **req_extensions = v3_req：此处忽略**。作用于 **CSR 生成阶段**（`openssl req`）。它将 `[ v3_req ]` 中的请求信息（我想要 SAN）打包进 `.csr` 文件。
- **-extensions v3_ext**：作用于 **证书签发阶段**（`openssl x509 -req`）。它告诉 CA（证书颁发机构）：“在最终签发的证书里，请应用 `[ v3_ext ]` 段落定义的扩展。”


> 特别注意：服务端证书、客户端证书、根证书， 三者的CN(Common Name)必须不一样，一样会有这个报错：error 18 at 0 depth lookup:self signed certificate。
>

```properties
cat << EOF | tee common.cnf
[ req ]
default_bits       = 2048
distinguished_name = req_distinguished_name
prompt             = no

[ req_distinguished_name ]
countryName                = GN
stateOrProvinceName        = Conakry
localityName               = Kaloum
organizationName           = Guinea Government
organizationalUnitName     = Root CA Department (NID)
commonName                 = Guinea National Root CA-PXC
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
openssl req -new -x509 -nodes -days 36500 -sha256 -key ca-key.pem -out ca.pem -config common.cnf
# 验证，查看CN 签名算法 sha256WithRSAEncryption
openssl x509 -in ca.pem -text -noout
openssl x509 -in ca.pem -text -noout | grep "Signature Algorithm"
```



## 三、服务端证书生成

### 1. 生成服务器密钥文件：server-key.pem 和 server-req.pem

```properties
vim common.cnf
# commonName 必须末尾添加 -server 最终: Guinea National Root CA-PXC-server
openssl req -newkey rsa:2048 -days 36000 -sha256 -nodes -keyout server-key.pem -out server-req.pem -config common.cnf
# 验证，查看CN 签名算法 sha256WithRSAEncryption
openssl req -in server-req.pem -text -noout
openssl req -in server-req.pem -text -noout | grep "Signature Algorithm"
```

### 2. 移除密码：server-key.pem

- 移除 加密的私钥文件开头 Proc-Type: 4,ENCRYPTED 

```properties
vim server-key.pem
md5sum server-key.pem
openssl rsa -in server-key.pem -out server-key.pem
md5sum server-key.pem
```

### 3. 生成 extfile文件

- **SAN中修改服务端的可用IP**

```properties
cat << EOF | tee san.cnf
[ v3_ext ]
subjectAltName = @alt_names

[ alt_names ]
# DNS.1 = gouvernement.gov.gn
# DNS.2 = *.gouvernement.gov.gn
IP.1  = 127.0.0.1
IP.2  = 10.10.20.201
IP.3  = 10.10.20.202
IP.4  = 10.10.20.203
EOF
```

### 4. 生成服务器证书文件：server-cert.pem

**根据 ca.pem + ca.pem + server-req.pem 三个文件生成 server-cert.pem**

```properties
openssl x509 -req -in server-req.pem -days 36000 -sha256 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
  -out server-cert.pem -extensions v3_ext -extfile san.cnf
# 验证，查看CN 签名算法 sha256WithRSAEncryption
openssl x509 -in server-cert.pem -text -noout 
openssl x509 -in server-cert.pem -text -noout | grep "Signature Algorithm"
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
# commonName 必须末尾添加 -peer 最终: Guinea National Root CA-PXC-peer
openssl req -newkey rsa:2048 -days 36000 -sha256 \
-nodes -keyout peer-key.pem -out peer-req.pem -config common.cnf
# 验证，查看CN 签名算法 sha256WithRSAEncryption
openssl req -in peer-req.pem -text -noout
openssl req -in peer-req.pem -text -noout | grep "Signature Algorithm"
```

### 2. 移除密码：peer-key.pem

- 移除 加密的私钥文件开头 Proc-Type: 4,ENCRYPTED 

```properties
vim peer-key.pem
md5sum peer-key.pem
openssl rsa -in peer-key.pem -out peer-key.pem
md5sum peer-key.pem
```

### 3. 生成 extfile文件

- **SAN中修改服务端的可用IP**

```properties
cat << EOF | tee san.cnf
[ v3_ext ]
subjectAltName = @alt_names

[ alt_names ]
# DNS.1 = gouvernement.gov.gn
# DNS.2 = *.gouvernement.gov.gn
IP.1  = 127.0.0.1
IP.2  = 10.10.20.201
IP.3  = 10.10.20.202
IP.4  = 10.10.20.203
EOF
```

### 4. 生成服务器peer证书文件：peer-cert.pem

**根据 ca.pem + ca.pem + peer-req.pem 三个文件生成 peer-cert.pem**

```properties
openssl x509 -req -in peer-req.pem -days 36000 -sha256 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
  -out peer-cert.pem -extensions v3_ext -extfile san.cnf
# 验证，查看CN 签名算法 sha256WithRSAEncryption
openssl x509 -in peer-cert.pem -text -noout 
openssl x509 -in peer-cert.pem -text -noout | grep "Signature Algorithm"
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
# commonName 必须末尾添加 -client 最终: Guinea National Root CA-PXC-client
openssl req -newkey rsa:2048 -days 36000 -sha256 \
-nodes -keyout client-key.pem -out client-req.pem -config common.cnf
# 验证，查看CN 签名算法 sha256WithRSAEncryption
openssl req -in client-req.pem -text -noout
openssl req -in client-req.pem -text -noout | grep "Signature Algorithm"
```

### 2. 移除密码：client-key.pem

- 移除 加密的私钥文件开头 Proc-Type: 4,ENCRYPTED 

```properties
vim client-key.pem
md5sum client-key.pem 
openssl rsa -in client-key.pem -out client-key.pem
md5sum client-key.pem 
```

### 3. 生成 extfile文件

- **SAN中修改客户端的可用IP**

```properties
cat << EOF | tee san.cnf
[ v3_ext ]
subjectAltName = @alt_names

[ alt_names ]
# DNS.1 = gouvernement.gov.gn
# DNS.2 = *.gouvernement.gov.gn
IP.1  = 127.0.0.1
IP.2  = 10.10.20.201
IP.3  = 10.10.20.202
IP.4  = 10.10.20.203
EOF
```

### 4. 生成客户端证书文件：client-cert.pem

```properties
openssl x509 -req -in client-req.pem -days 36000 -sha256 \
  -CA ca.pem -CAkey ca-key.pem -set_serial 01 \
  -out client-cert.pem -extensions v3_ext -extfile san.cnf
# 验证，查看CN 签名算法 sha256WithRSAEncryption
openssl x509 -in client-cert.pem -text -noout
openssl x509 -in client-cert.pem -text -noout | grep "Signature Algorithm"
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

### 6. windows设置环境变量

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































