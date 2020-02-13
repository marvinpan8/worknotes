## Docker 内容信任

### 创建授权密钥三种

- 这些密钥既可以使用本地`$ docker trust`生成，
- 也可以由证书颁发机构生成，
- 也可以从通用控制平面的 [客户端捆绑包中获取](https://docs.docker.com/engine/security/trust/ee/ucp/user-access/cli/#download-client-certificates)。

#### 一、使用Docker Trust生成密钥

Docker trust有一个用于委托密钥对的内置生成器 `$ docker trust generate <name>`。

**运行此命令将自动将委派私钥加载到本地Docker信任库。**

```bash
$ docker trust key generate marvinpan
Generating key for marvinpan...
Enter passphrase for new marvinpan key with ID 9deed25: 
Repeat passphrase for new marvinpan key with ID 9deed25: 
Successfully generated and loaded private key. Corresponding public key available: /etc/pki/ca-trust/extracted/pem/marvinpan.pub
```

#### 二、手动生成密钥

如果需要手动生成私钥（RSA或ECDSA）和包含公钥的x509证书，则可以使用本地工具（如openssl或cfssl）以及本地或公司范围的证书颁发机构。

以下是如何生成2048位RSA部分密钥的示例（所有RSA密钥必须至少为2048位）：

```bash
$ openssl genrsa -out delegation.key 2048
Generating RSA private key, 2048 bit long modulus
....................................................+++
............+++
e is 65537 (0x10001)
```

他们应该`delegation.key`保持私有化，因为它用于签署标签。

然后他们需要生成一个包含公钥的x509证书，这是你需要的。以下是生成CSR（证书签名请求）的命令：

```bash
$ openssl req -new -sha256 -key delegation.key -out delegation.csr
```

然后，他们可以将其发送给您信任的CA以签署证书，或者他们可以自签名证书（在此示例中，创建有效期为1年的证书）：

```bash
$ openssl x509 -req -sha256 -days 365 -in delegation.csr -signkey delegation.key -out delegation.crt
```

然后他们需要给你`delegation.crt`，无论是自签名还是CA签名。

最后，您需要将私钥添加到本地Docker信任库中。

```bash
$ docker trust key load delegation.key --name jeff
Loading key from "delegation.key"...
Enter passphrase for new jeff key with ID 8ae710e: 
Repeat passphrase for new jeff key with ID 8ae710e: 
Successfully imported key from delegation.key
```

#### 三、使用Universal Control Plane的客户端套件

通用控制平面（UCP）通过客户端软件包中生成的证书管理对其集群的CLI和API访问。这些证书和密钥可用作委托密钥对。在每个客户端捆绑包中，都有一个`key.pem`包含公钥（）的唯一私钥（）和x509证书`cert.pem`。

1）从[通用控制平面](https://docs.docker.com/engine/security/trust/ee/ucp/user-access/cli/#download-client-certificates)下载用户的客户端捆绑包 。

2）将客户端包解压缩到当前目录中

3）将私钥加载到本地Docker信任库中

```bash
$ docker trust key load key.pem --name jeff
Loading key from "key.pem"...
Enter passphrase for new jeff key with ID 9deed25: 
Repeat passphrase for new jeff key with ID 9deed25: 
Successfully imported key from key.pem
```

---

### 删除镜像内容信任签名

```
notary -s https://notary.harbor.test.investoday.net --tlscacert ~/.docker/tls/notary.harbor.test.investoday.net/ca.crt -d ~/.docker/trust remove -p harbor.test.investoday.net/jrtz/weather 0.1
```

