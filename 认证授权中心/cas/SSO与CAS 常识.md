# 前端需要了解的 SSO 与 CAS 知识

不管是什么公司，只要产品数量大于一个，那么单点登录势必是绕不过去的一个问题。作为前端程序员，我们对其虽然接触不多，但适当的了解还是必要的。本文就来谈谈单点登录相关的问题。

## 前置知识

了解 SSO，最好具备以下知识。当然，如果不是特别熟，也不影响阅读。

- cookie及session
- 浏览器同源策略及跨域
- 了解登录系统的构成

## 什么是 SSO 与 CAS?

### SSO

SSO 是英文 Single Sign On 的缩写，翻译过来就是单点登录。顾名思义，它把两个及以上个产品中的用户登录逻辑抽离出来，达到只输入一次用户名密码，就能同时登录多个产品的效果。

打个比方，SSO 和我们去迪士尼玩时购买的通票很像。![img](https://user-gold-cdn.xitu.io/2017/11/6/722924230df383cce6c9ec3ab8b1be48?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

我们只要买一次通票，就可以玩所有游乐场内的设施，而不需要在过山车或者摩天轮那里重新买一次票。在这里，买票就相当于登录认证，游乐场就相当于使用一套 SSO 的公司，各种游乐设施就相当于公司的各个产品。

![img](https://user-gold-cdn.xitu.io/2017/11/6/4a6f420786946c6054c9cdde40004acd?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



使用 SSO 的优点很明显：

- 提升用户体验。

就以我厂为例。我厂有两个产品，丁香人才网和丁香园论坛，假如你是我厂用户，肯定无法忍受登录丁香园论坛的时候输入一次用户名密码，登录人才网又要输入一次用户名密码吧？

- 避免重复开发

假如你是我厂后端，每天任务都饱和的不行，肯定无法忍受到人才网开发一套登录逻辑，到论坛又开发一套登录逻辑吧？

- 提升安全系数

假如你是我厂运维，发现了一个安全隐患需要紧急修复。你肯定无法忍受给茫茫多的产品后端都发一封邮件，责令修复吧？万一漏了一个呢？

综合看来，SSO 不仅是**有用**的，而且是**必要**的。

### CAS

SSO 仅仅是一种架构，一种设计，而 CAS 则是实现 SSO 的一种手段。两者是抽象与具体的关系。当然，除了 CAS 之外，实现 SSO 还有其他手段，比如简单的 cookie。

CAS （Central Authentication Service）中心授权服务，本身是一个开源协议，分为 1.0 版本和 2.0 版本。1.0 称为基础模式，2.0称为代理模式，适用于存在非 Web 应用之间的单点登录。**本文只涉及 CAS 1.0**，下文中将详细介绍。



## SSO 的演进与分类

下面详述一下各种场景下的 SSO，它们之间是逐步升级，逐步复杂化的关系。

### 1.同域 SSO

如图，同域 SSO 是最简单的一种情况。

此时，两个产品都是在一个域名下，单点登录是很自然的选择。我们来捋一捋步骤，搞清楚这里的步骤是理解后文的基础，千万不要跳过。

1.  用户访问产品 a，向 后台服务器发送登录请求。
2.  登录认证成功，服务器把用户的登录信息写入 session。
3.  服务器为该用户生成一个 cookie，并加入到 response header 中，随着请求返回而写入浏览器。该 cookie 的域设定为 dxy.cn。
4.  下一次，当用户访问同域名的产品 b 时，由于 a 和 b 在同一域名下，也是 dxy.cn，浏览器会自动带上之前的 cookie。此时后台服务器就可以通过该 cookie 来验证登录状态了。

实际上，这种场景就是最简单最传统的登录操作。虽然我们把产品 a 和 b 人为分开了，但由于它们在同域上，就算看成是同一产品的不同类目也未尝不可。我们没有设置独立的 SSO 服务器，因为业务后台服务器本身就足以承担 SSO 的职能。

![img](https://user-gold-cdn.xitu.io/2017/11/7/d5c406051e343391aa970e8960229356?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 2.同父域 SSO

同父域 SSO 是同域 SSO 的简单升级，唯一的不同在于，服务器在返回 cookie 的时候，要把cookie 的 domain 设置为其父域。

比如两个产品的地址分别为 a.dxy.cn 和 b.dxy.cn，那么 cookie 的域设置为 dxy.cn 即可。在访问 a 和 b 时，这个 cookie 都能发送到服务器，本质上和同域 SSO 没有区别。

### 3.跨域 SSO

可以看到，在上面两种情况下，我们都没有专门设置 SSO 服务器。但是当两个产品不同域时，cookie 无法共享，所以我们必须设置独立的 SSO 服务器了。这个时候，我们就是通过标准的 CAS 方案来实现 SSO 的。下面我们就来详细介绍一下：



## CAS 1.0 详解

CAS 1.0 协议定义了一组术语，一组票据，一组接口。

### 术语：

- Client：用户。
- Server：中心服务器，也是 SSO 中负责单点登录的服务器。
- Service：需要使用单点登录的各个服务，相当于上文中的产品 a/b。

### 接口：

- /login：登录接口，用于登录到中心服务器。
- /logout：登出接口，用于从中心服务器登出。
- /validate：用于验证用户是否登录中心服务器。
- /serviceValidate：用于让各个 service 验证用户是否登录中心服务器。

### 票据

- **TGT：**Ticket Grangting Ticket 

TGT 是 CAS 为用户签发的登录票据，拥有了 TGT，用户就可以证明自己在 CAS 成功登录过。**TGT 封装了 Cookie 值以及此 Cookie 值对应的用户信息**。当 HTTP 请求到来时，CAS 以此 Cookie 值（TGC）为 key 查询缓存中有无 TGT ，如果有的话，则相信用户已登录过。

- **TGC：**Ticket Granting Cookie

CAS Server 生成TGT放入自己的 Session 中，而 **TGC 就是这个 Session 的唯一标识（SessionId）**，以 Cookie 形式放到浏览器端，是 CAS Server 用来明确用户身份的凭证。

- **ST：**Service Ticket 

ST 是 CAS 为用户签发的访问某一 service 的票据。用户访问 service 时，service 发现用户没有 ST，则要求用户去 CAS 获取 ST。用户向 CAS 发出获取 ST 的请求，CAS 发现用户有 TGT，则签发一个 ST，返回给用户。用户拿着 ST 去访问 service，service 拿 ST 去 CAS 验证，验证通过后，允许用户访问资源。



票据之间的关系如下图。注意，PGTIOU, PGT, PT 是 CAS 2.0 中的内容，感兴趣的同学可以自行了解（在下面介绍）。

![img](https://user-gold-cdn.xitu.io/2017/11/7/2d8ebfb254dd30f0eeb92e811f5d0163?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## CAS 2.0 

### CAS代理模式

模式形式为用户访问App1，App1又依赖于App2来获取一些信息，如：User -->App1 -->App2 。

这种情况下，假设App2也是需要对User进行身份验证才能访问，那么，为了不影响用户体验（过多的重定向导致User的IE窗口不停地闪动)，CAS引入了一种Proxy认证机制，即CAS Client可以代理用户去访问其它Web应用。

代理的前提是需要CAS Client拥有用户的身份信息(类似凭据)。之前我们提到的TGC是用户持有对自己身份信息的一种凭据，这里的PGT就是CAS Client端持有的对用户身份信息的一种凭据。凭借TGC，User可以免去输入密码以获取访问其它服务的Service Ticket，所以，这里凭借PGT，Web应用可以代理用户去实现后端的认证，而**无需前端用户的参与**。

下面为代理应用（helloService）获取PGT的过程：（注：PGTURL用于表示一个Proxy服务，是一个回调链接；PGT相当于代理证；PGTIOU为取代理证的钥匙，用来与PGT做关联关系；）

![cas代理PGT获取](http://www.coin163.com/java/cas/images/cas_clip_image004.jpg) 

如上面的CAS Proxy图所示，**CAS Client** 在基础协议之上，在验证ST时提供了一个额外的PGT URL(而且是 SSL 的入口)给CAS Server，使得CAS Server可以通过PGT URL提供一个PGT给CAS Client。

CAS Client拿到了PGT(PGTIOU-85…..ti2td)，就可以通过PGT向后端Web应用进行认证。

下面是代理认证和提供服务的过程：

![img](http://www.coin163.com/java/cas/images/cas_clip_image005.jpg)



如上图所示，Proxy认证与普通的认证其实差别不大，Step1，2与基础模式的Step1,2几乎一样，唯一不同的是，Proxy模式用的是PGT而不是TGC，是Proxy Ticket（PT）而不是Service Ticket。

### 辅助说明

CAS的SSO实现方式可简化理解为：**1个Cookie和 N 个Session**。CAS Server创建cookie，在所有应用认证时使用，各应用通过创建各自的Session来标识用户是否已登录。

用户在一个应用验证通过后，以后用户在同一浏览器里访问此应用时，客户端应用中的过滤器会在session里读取到用户信息，所以就不会去CAS Server认证。如果在此浏览器里访问别的web应用时，客户端应用中的过滤器在session里读取不到用户信息，就会去CAS Server的login接口认证，但这时CAS Server会读取到浏览器传来的cookie（TGC），所以CAS Server不会要求用户去登录页面登录，只是会根据service参数生成一个Ticket，然后再和web应用做一个验证ticket的交互而已。


### 术语详解

CAS系统中设计了5中票据：TGC、ST、PGT、PGTIOU、PT。

- Ticket-granting cookie(TGC)：存放用户身份认证凭证的cookie，在浏览器和CAS Server间通讯时使用，并且只能基于安全通道传输（Https），是CAS Server用来明确用户身份的凭证；

- Service ticket(ST)：服务票据，服务的惟一标识码,由CAS Server发出（Http传送），通过客户端浏览器到达业务服务器端；一个特定的服务只能有一个惟一的ST；

- Proxy-Granting ticket（PGT）：由CAS Server颁发给拥有ST凭证的服务，PGT绑定一个用户的特定服务，使其拥有向CAS Server申请，获得PT的能力；

- Proxy-Granting Ticket I Owe You（PGTIOU）:  作用是将通过凭证校验时的应答信息由CAS Server 返回给CAS Client，同时，与该PGTIOU对应的PGT将通过回调链接传给Web应用。Web应用负责维护PGTIOU与PGT之间映射关系的内容表；

- Proxy Ticket (PT)：是应用程序代理用户身份对目标程序进行访问的凭证；

 

其它说明如下：

- Ticket Granting ticket(TGT)：票据授权票据，由KDC的AS发放。即获取这样一张票据后，以后申请各种其他服务票据(ST)便不必再向KDC提交身份认证信息(Credentials)；

- Authentication service(AS) ---------认证用服务，索取Credentials，发放TGT；

- Ticket-granting service (TGS) ---------票据授权服务，索取TGT，发放ST；

- KDC( Key Distribution Center ) ----------密钥发放中心；

## 详细步骤

看到这里，是不是又有点晕了？没关系，下面我们借助一个简单的场景，再来仔细捋一捋用 CAS 实现 SSO 的详细步骤，顺便加深理解之前提出的概念。

开始！

1. 用户访问产品 a，域名是 www.a.cn。

2. 由于用户没有携带在 a 服务器上登录的 a cookie，所以 a 服务器返回 http 重定向，重定向的 url 是 SSO 服务器的地址，同时 url 的 query 中通过参数指明登录成功后，回跳到 a 页面。重定向的url 形如 sso.dxy.cn/login?service=https%3A%2F%2Fwww.a.cn。

3. 由于用户没有携带在 SSO 服务器上登录的 **TGC**（看上面，票据之一），所以 SSO 服务器判断用户未登录，给用户显示统一登录界面。用户在 SSO 的页面上进行登录操作。

4. 登录成功后，SSO 服务器构建用户在 SSO 登录的 **TGT**（又一个票据），同时返回一个 http 重定向。这里注意：

5. - 重定向地址为之前写在 query 里的 a 页面。
   - 重定向地址的 query 中包含 sso 服务器派发的 **ST**。
   - 重定向的 http response 中包含写 cookie 的 header。这个 cookie 代表用户在 SSO 中的登录状态，它的值就是 **TGC**。

6. 浏览器重定向到产品 a。此时重定向的 url 中携带着 SSO 服务器生成的 **ST**。

7. 根据 **ST**，a 服务器向 SSO 服务器发送请求，SSO 服务器验证票据的有效性。验证成功后，a 服务器知道用户已经在 sso 登录了，于是 a 服务器构建用户登录 session，记为 a session。并将 cookie 写入浏览器。注意，此处的 cookie 和 session 保存的是用户在 a 服务器的登录状态，和 CAS 无关。

8. 之后用户访问产品 b，域名是 www.b.cn。

9.  由于用户没有携带在 b 服务器上登录的 b cookie，所以 b 服务器返回 http 重定向，重定向的 url 是 SSO 服务器的地址，去询问用户在 SSO 中的登录状态。

   

10. 浏览器重定向到 SSO。注意，第 4 步中已经向浏览器写入了携带 **TGC** 的cookie，所以此时 SSO 服务器可以拿到，根据 **TGC** 去查找 **TGT**，如果找到，就判断用户已经在 sso 登录过了。

11. SSO 服务器返回一个重定向，重定向携带 **ST**。注意，这里的 ST 和第4步中的 ST 是不一样的，事实上，每次生成的 ST 都是不一样的。

12. 浏览器带 **ST** 重定向到 b 服务器，和第 5 步一样。

13. b 服务器根据票据向 SSO 服务器发送请求，票据验证通过后，b 服务器知道用户已经在 sso 登录了，于是生成 b session，向浏览器写入 b cookie。

![img](https://user-gold-cdn.xitu.io/2017/11/7/0e2c2be35de3cf42744bae6f1238d6e7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



如图所示，至此，整个登录流程结束。之后当用户访问 a 或者 b 后，直接会携带 a cookie/b cookie，就不用再向 SSO 确认了。

实际开发时，可以根据 CAS 增加更多的判断逻辑，比如，在收到CAS Server签发的ST后，如果 ST 被 hacker 窃取，并且 client 本身没来得及去验证 ST，被 hacker 抢先一步验证 ST，怎么解决。此时就可以在申请 ST 时添加额外验证因子（如ip、sessionId等）。

# CAS安全性

CAS的安全性仅仅依赖于SSL。使用的是secure cookie。

### TGC/PGT安全性

对于一个 CAS 用户来说，最重要是要保护它的TGC，如果TGC不慎被CAS Server以外的实体获得，Hacker能够找到该TGC，然后冒充CAS用户访问**所有**授权资源。PGT的角色跟TGC是一样的。

从基础模式可以看出， TGC是CAS Server通过SSL方式发送给终端用户，因此，要截取TGC难度非常大，从而确保CAS的安全性。

**TGT的存活周期默认为120分钟。**

### ST/PT安全性

ST（Service Ticket）是通过Http传送的，因此网络中的其他人可以Sniffer到其他人的Ticket。CAS通过以下几方面来使ST变得更加安全（事实上都是可以配置的）：

**1、ST只能使用一次**

CAS协议规定，无论 Service Ticket验证是否成功， CAS Server都会清除服务端缓存中的该Ticket，从而可以确保一个Service Ticket不被使用两次。

**2、  ST在一段时间内失效**

CAS规定ST只能存活一定的时间，然后CAS Server会让它失效。默认有效时间为5分钟。

**3、  ST是基于随机数生成的**

ST必须足够随机，如果ST生成规则被猜出，Hacker就等于绕过CAS认证，直接访问**对应的**服务。

## 参考阅读：

- [CAS官方文档](https://link.juejin.im/?target=https%3A%2F%2Fapereo.github.io%2Fcas%2F4.2.x%2Findex.html)