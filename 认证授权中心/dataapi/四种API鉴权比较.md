## [WebAPI常见的鉴权方法，及其适用范围](https://www.cnblogs.com/mrbug/p/8027918.html)

在谈这个问题之前，我们先来说说在WebAPI中保障接口请求合法性的常见办法：

- API Key + API Secret
- cookie-session认证
- OAuth
- JWT

 当然还有很多其它的，比如 openid connect （OAuth 2.0协议之上的简单身份层），Basic Auth ，Digest Auth 不一一例举了

 

### 1、API Key + API Secret

Resource + API Key + API Secret 匹配正确后，才可以访问Resource，通常还会配合时间戳来进行时效控制。这种鉴权方式有以下特点：

1）API Key / API Secret这种模式本质是不是[RBAC](https://baike.baidu.com/item/RBAC/1328788?fr=aladdin)，而是做的[ACL](https://baike.baidu.com/item/ACL/362453?fr=aladdin)访问权限控制用的。

2）服务器负责为每个客户端生成一对 key/secret （ key/secret 没有任何关系，不能相互推算），保存，并告知客户端。

3）一般是把所有的请求参数（API Key也放在请求参数内）排序后和API Secret做hash生成一个签名sign参数，服务器后台只需要按照规则做一次签名计算，然后和请求的签名做比较，如果相等验证通过，不相等就不通过。

4）为避免重放攻击，可加上 timestamp 参数，指明客户端调用的时间。服务端在验证请求时若 timestamp 超过允许误差则直接返回错误。

5）一般来说每一个api用户都需要分配一对API Key / API Secret的，比如你有几百万的用户，那么需要几百万个密钥对的，数据量不大时一般存在xml中，数据量大时可以存在mysql表中。

 

#### 优点：

1）实现简单。

2）占用计算资源和网络资源都很少。

3）安全性较好。

 

#### 缺点：

1）当API Key / API Secret足够多时，服务端有一定的存储成本

2）鉴权本身不能承载其它的信息，服务端只能通过API Key来区别调用者。

3）API Secret一旦泄密，将是致命的。

 

#### 适用范围：

事实上，这种模式适用于大多数的WebAPI，除非你需要在token中承载更多的信息，或者你的API Key / API Secret足够多，多到能影响你服务端的布署。

 

### 2、cookie-session认证

这是比较老牌的鉴权方式了，这种鉴权方式有以下特点：

1）为了使后台应用能识别是哪个用户发出的请求，只能在后台服务器存储一份用户登陆信息，这份信息也会在响应前端请求时返回给浏览器（前端），前端将其保存为cookie。

2）下次请求时前端发送给后端应用，后端应用就可以识别这个请求是来自哪个用户了。

3）cookie内仅包含一个session标识符而诸如用户信息、授权列表等都保存在服务端的session中。

 

#### 优点：

1）老牌，资料多，语言支持完善。

2）较易于扩展，外部session存储方案已经非常成熟了（比如Redis）。

 

#### 缺点：

1）性能相于较低：每一个用户经过后端应用认证之后，后端应用都要在服务端做一次记录，以方便用户下次请求的鉴别，通常而言session都是保存在内存中，而随着认证用户的增多，服务端的开销会明显增大。

2）与REST风格不匹配。因为它在一个无状态协议里注入了状态。

3）CSRF攻击：因为基于cookie来进行用户识别, cookie如果被截获，用户就会很容易受到跨站请求伪造的攻击。有兴趣可以看下http://www.uml.org.cn/Test/201508124.asp 。

4）很难跨平台：在移动应用上 session 和 cookie 很难行通，你无法与移动终端共享服务器创建的 session 和 cookie。

 

#### 适用范围：

传统的web网站，且同时认证的人数不是足够大（是足够大）的都可以用这种方式，事实上，这种方式现在依旧在很各大网站平台上活跃着。

 

### 3、OAuth

OAUTH协议为用户资源的授权提供了一个安全的、开放而又简易的标准。与以往的授权方式不同之处是OAUTH的授权不会使第三方触及到用户的帐号信息（如用户名与密码），即第三方无需使用用户的用户名与密码就可以申请获得该用户资源的授权，因此OAUTH是安全的。oAuth是Open Authorization的简写。这种鉴权方式有以下特点：

1）OAuth在"客户端"与"服务提供商"之间，设置了一个授权层（authorization layer）。

2）"客户端"不能直接登录"服务提供商"，只能登录授权层，以此将用户与客户端区分开来。

3）"客户端"登录授权层所用的令牌（token），与用户的密码不同。用户可以在登录的时候，指定授权层令牌的 

​       权限范围和有效期。

4）"客户端"登录授权层以后，"服务提供商"根据令牌的权限范围和有效期，向"客户端"开放用户储存的资料。

 

#### 优点：

1）简单：不管是 OAUTH 服务提供者还是应用开发者，都很容易于理解与使用。

2）安全：没有涉及到用户密钥等信息，更安全更灵活。

3）开放：任何服务提供商都可以实现 OAUTH ，任何软件开发商都可以使用 OAUTH 。

 

#### 缺点：

1）需要增加授权服务器。

2）token载体信息单一，不利于服务端对调用者的统计等操作。

 

#### 适用范围：

常用于用户的授权，如使用QQ在其它网站上快速登录。很明显没有第三方参与的场景是不适合用 OAuth 的，这种情况可以使用API Key + API Secret或JWT。

 

关于OAuth的详细介绍，可以看大牛的文章 [《](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)[理](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)[解OAuth 2.0》](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

 

### 4、JWT

这种鉴权方式有以下特点：

1）JWT常常被用作保护服务端的资源（resource）。

2）客户端通常将JWT通过HTTP的Authorization header发送给服务端。

3）服务端使用自己保存的key计算、验证签名以判断该JWT是否可信。

4）在Web应用中，别再把JWT当做session使用，绝大多数情况下，传统的cookie-session机制工作得更好。

4）JWT适合一次性的命令认证，颁发一个有效期极短的JWT，即使暴露了危险也很小

5）由于每次操作都会生成新的JWT，因此也没必要保存JWT，真正实现无状态。

 

#### 优点：

1）易于水平扩展（当访问量足够在时，相对于cookie-session方案而言的。如果把session中的认证信息都保存在JWT中，在服务端就没有session存在的必要了。当服务端水平扩展的时候，就不用处理session复制（session replication）/ session黏连（sticky session）或是引入外部session存储了）

2）无状态。

3）支持移动设备。

4）跨程序调用。

5）安全。

6）token的承载的信息很丰富。

实事上，所有使用token的模式，都是无状态、跨程序调用的。



#### 缺点：

1）如果JWT中的payload的信息过多，网络资源占用较多。

2）JWT 的过期和刷新处理起来较麻烦。这个可以参考业界主流做法，AWS、Azure 和 Auth0 都是用 JWT 为载体，ID Token + Access Token + Refresh Token 的模式： 
https://docs.aws.amazon.com/zh_cn/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-with-identity-providers.html 
https://docs.microsoft.com/zh-cn/azure/active-directory/develop/active-directory-token-and-claims

 

#### 适用范围：

JWT（其实还有SAML）最适合的应用场景就是“开票”，或者“签字”。

例如：员工张三需要请假一天，于是填写请假条，张三在获得其主管部门领导签字后，将请假交给HR部门李四，李四确认领导签字无误后，将请假条收回，并在公司考勤表中做相应记录。这样张三就可以得到一天的假期，去浪。

在以上的例子中，“请假条”就是JWT中的payload（载荷就是存放有效信息的地方），领导签字就是base64后的数字签名（signature），领导是iss（issuer / jwt签发者），“HR部门的李四”即为JWT的aud（audience / 接收jwt的一方），aud需要验证领导签名是否合法，验证合法后根据payload中请求的资源给予相应的权限，同时将JWT收回。

 

### 总结

不要单纯的为了用某种鉴权方式而去强行使用，可以根据实际需要进行修改、扩展，或者是几个模式进行组合。总之，没有最好的，只有最适合的。