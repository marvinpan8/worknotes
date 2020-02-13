## Java开发企业级权限管理系统--Spring Security

### Spring Security - 优点

- 提供了一套安全框架，而且这个框架是可以用的
- 提供了很多用户认证的功能，实现相关的接口即可，节约大量开发工作
- 基于spring，易于集成到spring项目中，且封装了许多方法

### Spring Security - 缺点

- 配置文件多，角色被 “编码”到配置文件中和源文件中，RBAC不明显
- 对于系统中用户、角色、权限之间的关系，没有可操作的界面
- 大数据量情况下，几乎不可用

---
### 为什么需要权限管理

- 安全性：误操作、人为破坏、数据泄露

- 数据隔离：不同的权限能看到及操作不同的权限

- 明确职责：运营、客服等不同角色，leader和dev等不同级别

### 权限管理核心

- 用户-权限：人员少，功能固定，或者特别简单的系统

- RBAC（Role-Based Access Control）用户-角色-权限，都适用，遵循三个安全原则

  - 最小权限原则
  - 责任分离原则------角色互斥
  - 数据抽象原则

  ![1558060488292](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558060488292.png)

### 理想中的权限管理

- 能实现角色级权限：RBAC

- 能实现功能级、数据级权限

- 简单、易操作，能够应对各种需求

### 相关操作界面

- 权限管理界面、角色管理界面、用户管理界面
- 角色和权限关系维护界面、用户和角色关系维护界面

### Spring Security介绍

![1558062285580](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558062285580.png)

### Spring Security权限拦截器

![1558063401005](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558063401005.png)

### Spring Security数据库管理

![1558063477139](C:\Users\lenovo\AppData\Roaming\Typora\typora-user-images\1558063477139.png)

重要接口 UserDetailService----loadUserByUsername:  UserDetail ---安全用户信息源

重要对象： Authentication，认证对象（未认证，已认证），与UserDetail 比对，并拷贝权限集合

### Spring Security权限缓存

- CachingUserDetailsService
- EhCacheBasedUserCache

### Spring Security权限自定义决策

- 接口AccessDecisionManager ---> 抽象类AbstractAccessDecisionManager-->`supports`--->AccessDecisionVoter--->RoleVoter--->vote(Authentication authentication, Object object, Collection<ConfigAttribute> attributes)投票器

- 实现抽象类AbstractAccessDecisionManager有3个投票器

  - 投票器 AffirmativeBased -decide方法，**一票通过**

  - 投票器 ConsensusBased -decide方法，**需要一半及以上通过**

  - 投票器 UnanimousBased -decide方法，**全部必须通过**

  - 自定义投票器，继承父类AbstractAccessDecisionManager

  - 匹配两个以上权限时，需要自定义投票器AccessDecisionVoter的vote方法实现

---

