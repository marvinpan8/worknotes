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

**CMP** 的全称是 **Certificate Management Protocol**，即**证书管理协议**

| 特性           | CMP                                                          | SCEP                                                     |
| -------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| **功能丰富度** | 更全面，支持完整的证书生命周期管理（申请、颁发、吊销、更新、密钥恢复等）。 | 相对简单，主要侧重于证书的申请和颁发。                   |
| **安全性**     | 安全性更高，支持更复杂的消息保护和签名机制。                 | 安全性相对基础，但足以满足常见场景。                     |
| **适用场景**   | 对安全性和功能完整性要求高的企业级、大规模自动化环境。       | 更轻量级，常用于网络设备（如路由器、防火墙）的证书部署。 |

# EAC 1.11 传统 ePassport 只读终端

信任链为 `CVCA -> DV (domestic/foreign) -> Inspection System (IS)`。DV证书由CVCA直接签发



# EAC 2.10 新型可写终端

## ■ 认证终端 Authentication Terminal（AT）---可写

信任链为 CVCA → DV (DV_D 或 DV_F) → Authentication Terminal (AUTHTERM)。DV证书由CVCA直接签发。

认证终端是 **德国 eID 标准**（详见 **BSI_TR-03110_Part-4_V2-2** 第8页表格）。 认证终端可以对**电子身份证（eID）**上的任何**DG17~DG21** 数据组进行精细（读取和**写入**）控制，还可以控制其他功能：

- **PIN_MANAGEMENT**:  PIN 管理
- **AGE_VERIFICATION**:  年龄验证
- **COMMUNITY_ID_VERIFICATION**: 社区ID验证
- **RESTRICTED_IDENTIFICATION**：受限标识符     
- **PRIVILEGED_TERMINAL**    特权终端
- **CAN_ALLOWED**   允许CAN
- **INSTALL_CERT**   安装证书
- **INSTALL_QUALIFIED_CERT**  安装合格证书

## ■ 签名终端 Signature Terminal（ST）

信任链扩展为 `CVCA -> DV_AB -> DV_CSP -> SIGNTERM`。DV被进一步细分为具有不同职能的 `DV_AB` 和 `DV_CSP`，以支持更复杂的授权和管理流程。**电子签名（eSign）： ICAO芯片内部签名后返回给签名终端。**

### ■■ 角色Role

| 角色      | hex | 角色全称         | 职能与层级描述 (基于EAC2.10规范)                             |
| --------- | ---------- | --------------------- | ------------------------------------------------------------ |
| **CVCA**  | `0xC0`     | 国家证书验证机构      | **信任锚点**。每个国家/地区EAC证书体系的最高层根证书，负责签发下一级的DV证书。 |
| **DV_AB** | `0x80`     | **Accreditation Body:  **认证机构 | **中间层**。作为认证机构，是EAC信任链中负责对认证服务提供商进行审核和授权的权威实体。 |
| **DV_CSP**   | `0x40` | **Certification Service Provider:   **认证服务提供商 | **中间层**。作为认证服务提供商，是经过 **DV_AB** 授权，可以直接为终端（如签名终端）签发证书的实体。 |
| **SIGNTERM** | `0x00` | **Signature Terminal： **签名终端 | **终端层**。其证书由**DV_CSP**授权签发，EAC 2.10规范中定义的终端类型之一，其主要职能是生成具有法律效力的**电子签名**，而非仅读取数据。 |