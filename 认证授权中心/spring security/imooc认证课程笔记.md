## Spring Security开发安全的REST服务

### RESTful API

- 用URL描述资源
- 使用http方法描述行为。使用http状态码来表示不同的结果
- 使用json交互数据
- RESTful只是一种风格，并不是强制的标准

### @JsonView使用步骤

- 使用接口来声明多个视图
- 在值对象的get方法上指定视图
- 在Controller方法上指定视图

### 处理创建的请求

- @RequestBody 映射请求体到java方法的参数
- 日期类型参数的处理
- @Valid注解和BindResult验证请求参数的合法性并处理校验结果
- BindingResult是校验后需要处理时用的。

### 开发用户信息修改和删除服务

- 常用的验证注解-Hibernate Validator

- 自定义消息

- 自定义校验注解

### RESTful API的拦截

- 过滤器（Filter）--原始请求和响应
- 拦截器（Interceptor）--原始请求，处理方法信息
- 切片（Aspect）--处理方法信息，处理方法值，
- 请求顺序  Filter -->  Interceptor --> ControllerAdvice --> Aspect --> Controller

### 异步处理REST服务

- 使用Runnable(Callable)异步处理Rest服务
- 使用DeferredResult异步处理Rest服务
- 异步处理配置

### Spring security 核心功能

- 认证（你是谁）

- 授权（你能干什么）

- 攻击防护（防止伪造身份）

### Spring security 基本原理

![1557977668597](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\Spring security基本原理.png)

### 自定义用户认证逻辑

- 处理用户信息获取逻辑 `MyUserDetailsService implements UserDetailsService`
- 处理用户校验逻辑 `userdetails ``org.springframework.security.core.userdetails.User.User(String, String, boolean, boolean, boolean, boolean, Collection<? extends GrantedAuthority>)`
- 处理密码加密解密`org.springframework.security.crypto.password.PasswordEncoder`
  - encode 加密
  - matches 比对
### 个性化用户认证流程

- 自定义登录页面
  - http.formLogin().loginPage("/jrtz-signIn.html")
  - 修改登录路径 loginProcessingUrl("/authentication/form")
- 自定义登录成功处理 --- 实现 AuthenticationSuccessHandler
- 自定义登录失败处理 --- 实现 AuthenticationFailureHandler

### 系统配置封装

- SecurityProperties
  - BrowserProperties
  - ValidateCodeProperties
  - OAuth2Properties
  - SocialProperties

### 认证流程

![1558008118431](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558008118431.png)

#### 认证结果如何在多个请求之间共享

SecurityContextPersistenceFilter  设置和取Authentication

![1558008456739](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558008456739.png)

### 不同线程获取用户认证信息

方法一：

```bash
@GetMapping("/me")
public Object getCurrentUser() {
	return SecurityContextHolder.getContext().getAuthentication();
}
```
方法二：

```bash
@GetMapping("/me")
public Object getCurrentUser(Authentication authentication) {
	return authentication;
}
```

方法三：只获取 UserDetails 信息

```bash
@GetMapping("/me")
public Object getCurrentUser(@AuthenticationPrincipal UserDetails user) {
	return user;
}
```

## 图形验证码

### 生成图形验证码

- 根据随机数生成图形验证码
- 将随机数存到session中
- 再将生成的图片写到接口的响应中

### 重构图形验证码接口

- 验证码基本参数可配置
- 验证码拦截的接口可配置
- 验证码的生成逻辑可配置

#### 图形验证码基本参数配置

![1558014771757](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558014771757.png)

到了图形验证码重构