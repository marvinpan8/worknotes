# k8s部署ejbca-ca

- helm install my-ejbca-ce oci://repo.keyfactor.com/charts/ejbca-ce --version 9.3.7

```properties
kubectl create ns ejbca-ce
cd /k0s/ejbca
helm pull oci://repo.keyfactor.com/charts/ejbca-ce --version 9.3.7
tar -zxvf ejbca-ce-9.3.7.tgz

 # 执行本地安装
helm -n ejbca-ce install ejbca-ce /k0s/ejbca/ejbca-ce-9.3.7 -f ejbca-ce-values.yaml --wait
# 删除
# helm -n ejbca-ce delete ejbca-ce
```

CMP 的全称是 **Certificate Management Protocol**，即**证书管理协议**

| 特性           | CMP                                                          | SCEP                                                     |
| -------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| **功能丰富度** | 更全面，支持完整的证书生命周期管理（申请、颁发、吊销、更新、密钥恢复等）。 | 相对简单，主要侧重于证书的申请和颁发。                   |
| **安全性**     | 安全性更高，支持更复杂的消息保护和签名机制。                 | 安全性相对基础，但足以满足常见场景。                     |
| **适用场景**   | 对安全性和功能完整性要求高的企业级、大规模自动化环境。       | 更轻量级，常用于网络设备（如路由器、防火墙）的证书部署。 |

