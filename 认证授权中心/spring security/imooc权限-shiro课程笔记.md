# Java开发企业级权限管理系统--Shiro

### Apache Shiro 优点
- 提供了一套框架，而且这个框架可用，且易于使用
- 更灵活，应对需求能力强，Web能力强
- 可以与很多框架和应用集成
### Apace Shiro 缺点
- 学习资料少一些
- 除了自己实现RBAC外，操作的界面也需要自己实现

---
框架图

![1558081570023](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558081570023.png)

![1558082943815](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558082943815.png)

架构图

![1558082988889](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558082988889.png)

身份认证

![1558083433808](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558083433808.png)shiro授权

![1558095153619](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558095153619.png)

#### 权限校验流程

- WildcardPermissionResolver ---> WildcardPermission (*  :  ,) ---> AuthorizationInfo ---> getObjectPermissions() ---> getStringPermissions() --->PermissionResolver 解析为Permission实例，获取用户的角色 --->RolePermissionResolver解析权限对应的角色集合（自己实现）--->Permission.implies()逐个与权限比较，返回true or false

shiro 权限拦截

![1558096770530](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558096770530.png)

过滤器关系图

![1558098177410](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558098177410.png)

session管理器

![1558098479876](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558098479876.png)

shiro 缓存---主要为Realm 和Session缓存

![1558098561510](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558098561510.png)

fadsf

![1558098883055](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558098883055.png)

SessionManager

![1558099021282](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558099021282.png)

#### 重要的类

org.apache.shiro.web.filter.mgt.DefaultFilter

校验角色名称 org.apache.shiro.web.filter.authz.RolesAuthorizationFilter

校验权限名称 org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter