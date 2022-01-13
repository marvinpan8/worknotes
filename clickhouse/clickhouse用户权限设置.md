# clickhouse 用户权限设置

https://www.jianshu.com/p/3e08a7150fb1?hmsr=toutiao.io

> 使用clickhouse多半应用在实时数仓项目来支持 adhoc 查询，为了确保企业数据安全高效的使用，那么权限控制与资源隔离是必不可少的

[clickhouse最佳实践](https://www.jianshu.com/p/a72a4782a102)
 [clickhouse高级功能上线之mysql实时数据同步](https://www.jianshu.com/p/d0d4306411b3)

[clickhouse如何构建复杂数据模型](https://www.jianshu.com/p/7b2f17ef4ab7)
 [clickhouse sql规范](https://www.jianshu.com/p/e5050f046699)

clickhouse在20.4之后的版本开始支持基于RBAC的访问控制管理；主要包括的功能有：用户创建、角色创建、权限管理以及资源隔离；接下来我们将演示如何使用这些功能

### 配置管理员账户

管理员账户主要用来进行权限分配和管理用的；需要在user.xml中进行如下配置：

```xml
<users>
      <admin>  ## clickhouse自带default用户，但是该用户拥有所有权限且没有设置登陆密码和开启RBAC
        <password>pwd</password>
        <access_management>1</access_management>
        <networks incl="networks" replace="replace">
                <ip>::/0</ip>
        </networks>
        <profile>default</profile>
        <quota>default</quota>
      </admin>
</users>
```

- **access_management** 默认为0，设置为1标识开启RBAC权限控制。
- **admin** 配置的用户名
- **password** 用户对应的密码
- **networks** 运行访问的客户端ip、host
- **profile** clickhouse角色
- **quota** 配额，分配给该用户的资源

配置 config.xml 新增如下配置，用来存储创建的用户和角色

```ruby
## 用来存储创建的用户和角色
<access_control_path>/data/work/clickhouse/access/</access_control_path>

## 此配置必须有 否则会报
## DB::Exception: Not found a storage to insert user
```

### 创建普通用户

普通用户是权限作用的实体，可以设置独立的密码和操作范围，业务人员通过登陆不同的账户来行使不同的数据权限。

- create user

```php
## 创建用户u1 并限制ip只能在本地登陆数据库
## 通过明文或加密方式配置密码
## 创建时赋予了除role1、role2外的所有角色
CREATE USER u1 HOST IP '127.0.0.1' IDENTIFIED WITH PLAINTEXT_PASSWORD BY '123' DEFAULT ROLE ALL EXCEPT role1, role2


VM_10_14_centos :) show users;

SHOW USERS

┌─name────┐
│ admin   │
│ default │
│ u1      │
└─────────┘
```

- alter user

```bash
## 将用户名u1修改成u2  
ALTER USER u1 RENAME TO u2 
```

- drop user

```css
DROP USER [IF EXISTS] name [,...] [ON CLUSTER cluster_name]
```

- show create user
   查看用户创建语句

### 创建角色

角色是权限的集合，用来定义用户行使权限的范围。
 如果表被删除，和这张表关联的权限不会被删除。这意味着如果你创建一张同名的表，所有的特权仍旧有效。如果想删除这张表关联的特权，你可以执行 REVOKE ALL PRIVILEGES ON db.table FROM ALL 查询。

- create role

```undefined
CREATE ROLE r_reade_all SETTINGS max_memory_usage = 5000000 READONLY
```

- alter role

```bash
## 修改角色名r1为r2
alter role r1 rename to r2

## 如果是修改setting部分，需要重新登录才会生效
## 将内存改为50G
ALTER ROLE r_reade SETTINGS max_memory_usage = 50000000000, max_threads = 2
```

- drop role

```css
DROP ROLE [IF EXISTS] name [,...] [ON CLUSTER cluster_name]
```

### 权限管理

- 授权

```csharp
##将查询权限付给角色role1，然后把角色分配给用户u1

GRANT SELECT(x,y) ON db.* TO role1 WITH GRANT OPTION;

GRANT [ON CLUSTER cluster_name] role1 TO u1 [WITH ADMIN OPTION]
```

- 解除授权

```csharp
REVOKE SELECT(wage) ON accounts.staff FROM mira;
## or
REVOKE role1  FROM mira;
```

- 行级权限

```bash
## 授权给角色
CREATE ROW POLICY filter ON mydb.mytable FOR SELECT USING a<1000 TO r1, r2

## 授权给用户
CREATE ROW POLICY filter ON mydb.mytable FOR SELECT USING a<1000 TO ALL EXCEPT u1
```

## 资源限制

通过该特性，可快速实现clickhouse的存算分离

- cpu & memory

```undefined
CREATE SETTINGS PROFILE max_memory_usage_profile SETTINGS max_memory_usage = 102400 ,max_threads = 3  TO r1/u1
```

- 限制单位时间内查询次数

```bash
## 允许用户u1一天内最大只能发起20次查询
CREATE QUOTA qA FOR INTERVAL 1 DAY MAX QUERIES 20 TO r1/u1
```

## 总结

细粒度的权限控制和资源隔离将极大提升大数据人员的开发效率，通过开放数据和工具的方式赋能上层业务，上层业务不用在排队烧香和数据团队对需求了，整个流程得到了极大简化；每一个优秀的工程师都将有机会成为优秀的数据分析师