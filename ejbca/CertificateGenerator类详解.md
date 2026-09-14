# CertificateGenerator 类详解

| 方法签名     | 作用说明     |
| --------------| -------------- |
| `createTestCertificate(...)`   | **测试专用**：生成有效期3个月、哈希算法为 `SHA1withRSA`、授权角色为 `IS` 的测试证书。**（注释建议移至测试用例）** |
| `createCertificate(PrivateKey, String, CVCertificateBody, String)` | **核心方法**：基于已构建的证书体 `CVCertificateBody`，使用指定签名算法和 Provider 进行签名，生成完整的 `CVCertificate`。 |
| `createCertificate(PublicKey, PrivateKey, String, CAReferenceField, HolderReferenceField, AuthorizationRole, AccessRights, Date, Date, Collection<CVCDiscretionaryDataTemplate>, String)` | **完整证书生成**：根据公钥、私钥、算法、CA引用、持有者引用、授权角色、访问权限、有效期、扩展字段和 Provider，构造证书体并签名生成证书。 |
| `createCertificate(PublicKey, PrivateKey, String, CAReferenceField, HolderReferenceField, AuthorizationRole, AccessRights, Date, Date, String)` | 上述方法的重载，**不包含扩展字段**（扩展传 `null`）。        |
| `createCertificate(PublicKey, PrivateKey, String, CAReferenceField, HolderReferenceField, AuthorizationRoleEnum, AccessRightsIS, Date, Date, String)` | **二进制兼容性重载**：将 `AuthorizationRoleEnum` 和 `AccessRightsIS` 转换为父接口类型，再调用完整方法。 |
| `createRequest(KeyPair, String, HolderReferenceField)`       | 生成**无外签名**的 CVC 请求（使用 BouncyCastle 作为 Provider），仅含持有者引用。 |
| `createRequest(KeyPair, String, HolderReferenceField, String)` | 同上，但允许指定签名 Provider。                              |
| `createRequest(KeyPair, String, CAReferenceField, HolderReferenceField)` | 生成 CVC 请求，**包含 CA 引用**，使用 BouncyCastle。|
| `createRequest(KeyPair, String, CAReferenceField, HolderReferenceField, String)` | 同上，但允许指定签名 Provider。|
| `createRequest(KeyPair, String, CAReferenceField, HolderReferenceField, Collection<CVCDiscretionaryDataTemplate>, String)` | **最完整请求生成**：包含 CA 引用、持有者引用、扩展字段，并允许指定 Provider。内部构造请求体并签名（证明持有私钥）。 |
| `createAuthenticatedRequest(CVCertificate, KeyPair, String, CAReferenceField)` | 生成**经过认证的 CVC 请求**（含外签名），使用 BouncyCastle 作为 Provider。`caRef` 应为请求中 CA 引用序列号递增后的值。 |
| `createAuthenticatedRequest(CVCertificate, KeyPair, String, CAReferenceField, String)` | 同上，但允许指定签名 Provider。生成 `CVCAuthenticatedRequest` 对象，包含内部请求体和外部签名。 |

## ■ 比较 createCertificate 和 createRequest 

| 对比维度     | createCertificate (生成证书)                                 | **createRequest (生成请求)**                                 |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **核心用途** | 颁发一个正式的 **CVC 证书**，用于身份认证和授权。            | 生成一个 **证书请求** (类似于 PKCS#10)，用于向 CA 申请证书。 |
| **有效期**   | **有** (`validFrom`, `validTo` 参数)。证书有明确的时间范围。 | **无**。请求只是一个申请意向，不包含时间信息。               |
| **必须参数** | 需要 **CA 私钥** (`signerKey`) 来签名，代表 CA 的权威背书。  | 只需要 **终端实体自己的密钥对** (`keyPair`)，用其私钥证明对该公钥的持有权（Proof of Possession）。 |
| **书体内容** | 包含**授权角色** (`AuthorizationRole`)、**访问权限** (`AccessRights`) 等完整属性。 | **没有**授权角色和访问权限，仅包含公钥和持有者信息（简化版 `CVCertificateBody`）。 |
| **签名者**   | **CA** 使用其私钥对证书体进行签名。                          | **请求者自己**使用其私钥对请求体进行签名（证明拥有对应的私钥）。 |
| **生成对象** | `CVCertificate` (最终证书)。                                 | `CVCertificate`（实际上就是一个请求对象，内部标志不同）。    |
| **扩展字段** | 完整方法支持传入 `extensions` (如 `Certificate Extensions`)。 | 完整方法也支持 `extensions`，但通常用于请求特定扩展。        |

## ■ 比较 createRequest 和 **createAuthenticatedRequest** 

- **createRequest** 可能用于 **CVCA（国家根 CA）** 为 **DV（终端验证机构）** 签发证书前的初始请求。
- **createAuthenticatedRequest** 常用于 **终端（如电子护照）** 向 **DV** 或 **IS（检查站）** 发起认证时的请求，因为需要 CA 层级的事先授权，确保请求的合法性和不可否认性。

| 对比维度     | **createRequest (普通请求)**          | **createAuthenticatedRequest (认证请求)**             |
| ------------ | -------------- | ------------------- |
| **生成对象** | `CVCertificate` (实际是 `CVCRequest`) | `CVCAuthenticatedRequest` (是 `CVCertificate` 的子类) |
| **签名层数** | **单层签名**（内签名）  | **双层签名**（内签名 + 外签名） |
| **内签名** | 请求者用自己的私钥对请求体签名，证明**拥有该公钥对应的私钥**（Proof of Possession）。 | 与 `createRequest` 完全一样，请求者先对自己的请求体进行签名。 |
| **外签名** | **无**                                                       | **有**。CA（或上级机构）使用自己的私钥对整个请求（包含内签名）再次签名，表示**“该请求已被 CA 授权/认可”**。 |
| **签名参数**       | 只需要请求者的 `KeyPair`。                                   | 需要请求者的 `KeyPair`（用于内签名）+ **另外的 CA 密钥对**（用于外签名，作为方法参数传入）。 |
| **参数中的 caRef** | `caRef` 是可选的，只是请求体中的一个字段（表明希望由哪个 CA 签发）。 | `caRef` **必须**提供，且官方注释特别指出：**应为请求中 CA 引用序列号递增后的值**（用于区分不同的认证请求）。 |
| **适用场景** | 终端实体向 CA 提交**初次申请**证书。 | 用于**安全通道建立**或**交互式认证**流程（如 EAC 协议中的终端认证），需要 CA 事先对请求进行背书。 |
| **类比**     | 类似于 PKCS#10 证书请求文件。        | 类似于一个经过 CA“**预批准**”或“**公证**”过的请求，可以放心使用。 |

### ■■ 图示理解（签名结构）

- **createRequest 产生的结构：

  ```properties
  [ 请求体 (含公钥、持有者信息) ] --请求者私钥签名--> 内签名
  ```

- **createAuthenticatedRequest 产生的结构：**

  ```properties
  [ 请求体 (含公钥、持有者信息) ] --请求者私钥签名--> 内签名
  将上述整体 --CA 私钥签名--> 外签名（包在最外层）
  ```