# [前后端分离下的CAS跨域流程分析](https://segmentfault.com/a/1190000015235402)

## 写在最前

### 前后端分离其实有两类：

1. 开发阶段使用dev-server，生产阶段是打包成静态文件整个放入后端项目中。
2. 开发阶段使用dev-server，生产阶段是打包成静态文件放入单独的静态资源服务器中，如nginx。

这两种方案最大的区别就是生产阶段。由于第一种方案前端和后端本质在同一个服务中的，所以压根就没有跨域，配置cas的坑比较少。而第二种方案我们一般使用nginx反向代理完成跨域，配置cas的坑会很多。为了后面分析方便，我们分别称上述两种方案为『前后端分离A』和『前后端分离B』

### 请求也分为两类：

1.HTTP请求：像浏览器地址栏发起的请求、浏览器自发的访问某个网址、Postman测试接口，这些行为其实都是发起的HTTP请求，不会有跨域问题。

2.AJAX(XMLHttpRequest)请求：这是浏览器内部的XMLHttpRequest对象发起的请求，浏览器会禁止其发起跨域的请求，主要是为了防止跨站脚本伪造的攻击(CSRF)。

## 难点分析

前后端分离、跨域、CAS这三项技术单独使用起来，甚至拿其中两个出来一起使用，难度都不大，下面来列举一下：

1. 前后端分离(AB)+跨域
2. 前后端分离A+CAS（因为A方案根本就没有跨域这一说）
3. 前后端分离B+跨域+CAS

### 前后端分离(AB)+跨域

这个最简单，只有跨域，没有CAS，常见的CORS、反向代理、JSONP都可以解决

### 前后端分离A+CAS

#### 坑：CAS认证过期，莫名出现跨域错误的问题

可能有人会问，刚才不是说方案A压根就没有跨域问题吗？其实，这个跨域错误不怪我们的后端，而是怪CAS那边的后端，待我详细说来。

正常情况下，CAS认证成功后，浏览器会设置好一个来自CAS的Cookie以维持与CAS的Session。之后每次请求，无论是ajax请求还是http请求，都会带上这个cookie。而我们自己的后端服务器也会有一个CAS Authorization的过滤器，把没有CAS认证过的请求重定向到CAS的login页面。因为本次请求我们带了cas的cookie，所以请求顺利通过filter来到controller层，进而返回数据。

但是考虑这样一个情况，今天你打开你的浏览器，访问一个你们新做的cms系统的网址，然后跳到cas login页面，正常登陆，正常使用。然后来到第二天早上，因为昨天的页面你没关，你直接点了一个查询按钮，结果报错了。你打开浏览器控制台，竟然发现报了一个跨域的错误。这里有两处困惑：

1. 为什么他喵的会跨域呢？我们前后端命名部署在一台服务器上，是同域的啊。
2. 为什么cas认证失效后，没有自动跳到cas登录页呢？可是我之前直接在浏览器输入cms系统的地址时，因为没认证过，浏览器是能直接跳到cas登录页的，为什么这次不行呢？
3. ajax到底怎么处理302的

**为什么会跨域：**
想想一下这样的一个流程：第二天早上你来，点击一个查询按钮，发起了ajax请求，请求中带上了一个已经失效的cookie，然后请求被后端cas filter拦截，发现已失效，让后302跳转到cas login界面。**在这个过程中，你之前发起的ajax请求其实被redirect到了cas的login.html页面（这只是表象，本质后面会提到）**。你相当于发起ajax请求去请求一个html文件下来，然而cas的服务器并没有配置跨域，为了安全考虑也不能配置跨域，所以你的ajax请求还没来得及请求下来数据，你就被浏览器认为是跨域了，因为你的确在请求cas服务器的一个静态资源。

退一万步说，就算cas服务器配置了跨域，虽然你点击查询按钮的行为不会报跨域错误了，但你依然不能自动跳转到cas login页面，因为这个login.html直接当做你ajax的success中的回调参数回来了，浏览器是不会帮你跳转的。

**为什么不能跳转:**
首先，你打开浏览器输入cms系统的地址去访问的时候，发起的是HTTP请求，是不存在跨域问题的。因此你的HTTP请求被后端的filter给redirect到了cas的login.html，这个流程是没问题的。而你点击查询按钮，发起的是ajax请求，是没法跳转的**（具体原因见下方文字）**

**ajax在302中的行为本质**
当你点击查询按钮，发起的是ajax请求，请求被后端filter拦截，并告知你302跳转到login页，此时浏览器首先会感知到这次ajax请求的302状态，并替ajax去访问要跳转到的地址，然后将**访问的结果（其实就是整个login.html页面）**返回到你的ajax的success回调函数中，因此这个回调函数的参数其实就是整个login.html的页面。并且，直到浏览器把html放到ajax的success回调函数后，ajax才会真正的回调，之前的302状态ajax是感知不到的，当然也获取不到，所以**想通过ajax判断status是否是302，进而手动location.href到login页的方案是不行的**。

其实，这么看起来就像是你的ajax直接请求到了login.html页面。
另外，在实际cas跳转的过程中，在ajax的success回调之前，你的ajax操作就被浏览器认为是跨域了，所以你压根就没机会回调success，也因此获取不到status状态或者那个没卵用的login.html。

#### 好了，疑惑解决完了，该说说解决方案了：

我们要实现的就是：在cookie失效时，点击查询按钮后，能自动跳转到cas登录页。
方案很多，但都靠一下两点：
**用HTTP请求替代Ajax请求去跳转到登录页**
**用200代替302告知ajax当前请求的状态**

举几个例子：

1、错误方案：设法拦截ajax的response，然后判断response的status是否是302，如果是302就手动location.href跳到cas登录页，但是这样是不行的，因为我们根本获取不到这个302状态。
2、必须要后端配合，后端需要额外加1个filter和1个controller, 起个名字吧，就叫ValidateFilter和ValidateController吧。

ValidateFilter只过滤那些需要被cas拦截的请求，在doFilter里面判断HttpServletRequest的状态，看看这个request里能不能获取到当前用户名，如果能获取到，代表认证没问题，让这个请求继续往下走chain.doFilter，如果不能获取到，代表认证失效了（因为filter不能直接返回，所以我们需要一个ValidateController），我们request.dispatch这个请求到ValidateController的redirect方法中（自己写的），让这个redirect方法返回一个result，result中设置一个标志，比如给code:xxx。

然后前端设法在ajax的response之前获取response的result，看看result的code是否为xxx，如果是，那就location.href跳转到cas登录页即可，**其中service参数写cas登陆之后要回调的后端接口，然后让后端去跳转到前端页面。**

为什么不能直接service写前端？
因为我们不仅要跟cas服务器维持session，还要跟我们自己的后端维持session，如果不回调后端，后端就不会感知到我们的登录状态了。

比如:

```java
//前端:
if(result.code === xxx) {
    location.href = "http://cas.server.com/login?service=http://后端服务器地址/redirect/to/frontend?currentPath=当前页面路径"
    //currentPath是为了login之后再调回当前页面
}
//后端 filter 伪代码:
void doFilter(request, response, chain) {

    if(request中有用户名) {
      chain.doFilter()
    } else if(request.uri == '/redirect/to/caslogin') {
      chain.doFilter()
    } else {
        request.dispatch("/redirect/to/caslogin")
    }
}
//后端 controller 伪代码
// 用来接受filter过来的那些认证失效的请求
@path("/redirect/to/caslogin")
String redirectToCasLogin(request, response) {
    return {
        "code": xxx
    }
}
// 用来在login之后回调用
@path("/redirect/to/frontend")
String redirectToFrontend(request, response) {
  String path = request中的currentPath参数
  response.sendRedirect(path)
}

// 另外，这个controller一定不要被validateFilter过滤，因为如果这个controller也要被过滤，那就陷入cas验证的死循环了。
```

3.和2类似，但是location.href中直接写

```js
location.href = "http://后端服务器地址/redirect/to/caslogin?currentPath=当前页面路径"
```

此时我们直接请求后端接口/redirect/to/caslogin，他首先被validateFilter拦截，但是因为有一个if判断，他被直接doFilter，然后请求来到了cas的Filter，因为没登录，该filter会自动拼接我们配置的cas serverName+当前请求的uri，同样会形成
http://cas.server.com/login?service=http://后端服务器地址/redirect/to/frontend?currentPath=当前页面路径"这样的url

### 前后端分离B+跨域+CAS

写不动了，总之要注意：要保持cookie的域一致

对于nginx，如果从 www.a.com/ 代理到 www.b.com/api，那么形成的cookie的域是会是/api，而浏览器发起请求时只能携带/域的cookie，所以导致cookie丢失，session失效。可以通过nginx配置，把/api域下的cookie都放到/即可解决。
为了避免额外的麻烦，最好保持代理前后url一致吧，即都有一个/api前缀，或者都没有。

对于浏览器，发起的ajax所带的cookie是发起请求的host域名有严格关系的，不同的域名带不同的cookie，所以如果出现，你明明已经登陆了，但是在此发起ajax请求，后端还是识别不出来你的登录状态，那就可能是你发起的请求的域名不一致了。也就是说，你去请求后端接口的时候用www.a.com，结果cas登陆成功后的要回调的接口成了www.b.com，这样你的cas登录状态的cookie就附着在www.b.com的域名上了，然后当你再发起www.a.com的请求的时候，发现你根本带不上cas下来的cookie，因为域不同。
这种情况通常发生在反向代理的时候，前端发起ajax请求代理服务器www.a.com，代理服务器发起请求到www.b.com，这时候就容易导致域名不一致，请一定要注意这点。

**另外，对于当前前后端分开部署的情况，location.href中，service的回调接口不能直接写后端地址（相当于www.b.com），而应该写www.a.com，让代理服务器去访问www.b.com，这样才能保持cookie的域的一致性！！！！**