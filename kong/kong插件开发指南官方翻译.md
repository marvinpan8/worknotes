# kong插件开发指南官方翻译

<https://www.cnblogs.com/huxianglin/p/7263964.html>

## 目录

1.  介绍
2.  文件结构
3.  编写自定义逻辑
4.  存储配置
5.  访问数据存储
6.  自定义实体
7.  缓存自定义实体
8.  扩展Admin API
9.   编写测试
10. （卸载）安装你的插件

## 插件开发 - 介绍

### 什么是插件，他们如何与kong集成？

在进一步之前，有必要简要解释一下如何构建，特别是它如何与Nginx集成，以及Lua与它有关。

`lua-nginx-module`模块可以在Nginx中启用Lua脚本功能。 Kong并没有使用这个模块编译Nginx，而是与OpenResty一起发行，OpenResty已经包含了`lua-nginx-module`。OpenResty不是Nginx的分支，而是一系列扩展其功能的模块。

因此，Kong是一个Lua应用程序，旨在加载和执行Lua模块（我们更常称之为“插件”），并为它们提供了一个完整的开发环境，包括数据库抽象，迁移，帮助等等...

您的插件将由Lua模块组成，将由Kong加载和执行，它将受益于两个API：

- [lua-nginx-module API](https://github.com/openresty/lua-nginx-module)：允许与Nginx本身进行交互，例如检索请求/响应或访问Nginx的共享内存区域。
- Kong的插件环境：允许与保存配置的数据存储（API，消费者，插件...）和各种帮助器进行交互，从而允许插件之间的交互。这是本指南将要描述的环境。

```
注意：本指南假设您熟悉Lua和lua-nginx-module API，并且只会描述Kong的插件环境。
```

## 插件开发 - 文件结构

```
注意：本章假设您熟悉Lua。
```

将您的插件视为一组Lua模块。本章中描述的每个文件都将被视为单独的模块。如果他们的名字符合这个约定，Kong会检测并加载您的插件的模块：

```
"kong.plugins.<plugin_name>.<module_name>"
```

**您的模块当然需要通过您的 package.path 变量来访问，可以通过您的Nginx配置中的lua-package-path指令来调整您的需求。然而，安装插件的首选方法是通过Luarocks。在本指南中的更多内容。**

为了让Kong知道它必须找到你的插件的模块，你必须将它添加到你的配置文件中的`custom_plugins`属性中。例如：

```
custom_plugins:
  - my-custom-plugin # your plugin name here
```

现在，Kong将尝试加载本章所述的模块。其中有些是强制性的，但不会被忽略，而Kong会认为你不会使用它。例如，Kong将加载`“kong.plugins.my-custom-plugin.handler”`来检索和执行插件的逻辑。

现在让我们来描述你可以实现什么模块以及它们的目的。

------

### 基本插件模块

在最基本的形式中，一个插件由两个必需的模块组成：

```
simple-plugin
├── handler.lua
└── schema.lua
```

- handler.lua：你的插件的核心。它是一个实现的接口，其中每个功能将在请求的生命周期中的所需时刻运行。
- schema.lua：您的插件可能必须保留用户输入的一些配置。该模块保存该配置的模式并定义其上的规则，使用户只能输入有效的配置值。

------

### 高级插件模块

一些插件可能必须更深入地与Kong集成：在数据库中拥有自己的表，在Admin API中公开端点等...每个都可以通过在你的插件中添加一个新的模块来完成。这是一个插件的结构，如果它正在实现所有可选的模块：

```
complete-plugin
├── api.lua
├── daos.lua
├── handler.lua
├── hooks.lua
├── migrations
│   ├── cassandra.lua
│   └── postgres.lua
└── schema.lua
```

以下是实现可能的模块的完整列表，并简要说明其目的。本指南将详细介绍，让您掌握他们每一个。

| 模块名称         | 需要 | 描述                                                         |
| ---------------- | ---- | ------------------------------------------------------------ |
| api.lua          | No   | 定义在Admin API中可用的端点列表，以便与您的插件处理的实体自定义实体进行交互。 |
| daos.lua         | No   | 定义DAO（数据库访问对象）的列表，它是插件所需的存储在数据存储中的自定义实体的抽象。 |
| handler.lua      | Yes  | 一个需要被实现的接口。每个功能将由Kong在请求的生命周期中的所需时刻运行。 |
| migrations/*.lua | No   | 给定数据存储区的相应迁移。只有当您的插件必须将自定义实体存储在数据库中并通过daos.lua定义的DAO之一与之进行交互时，才需要进行迁移。 |
| hooks.lua        | No   | 对daos.lua中定义的数据存储实体实现无效事件处理程序。如果要将实体存储在内存中的缓存中，以便在数据存储上进行更新/删除时使其无效，则为必需。 |
| schema.lua       | Yes  | 保存插件配置的架构，以便用户只能输入有效的配置值。           |

------

## 插件开发 - 编写自定义逻辑

### 模块

```
"kong.plugins.<plugin_name>.handler"
```

------

```
注意：本章假设您熟悉Lua和lua-nginx-module API。
```

Kong允许您在请求的生命周期中的不同时间执行自定义代码。为此，您必须实现`base_plugin.lua`接口的一个或多个方法。这些方法将在一个模块中实现：`“kong.plugins”<plugin_name> .handler“`

------

### 可用请求上下文

Kong允许您在所有lua-nginx模块上下文中编写代码。当您的请求达到上下文时，将在执行的`handler.lua`文件中执行每个函数：

| 函数名             | LUA-NGINX-MODULE 背景                                        | 描述                                                         |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `:init_worker()`   | [init_worker_by_lua](https://github.com/openresty/lua-nginx-module#init_worker_by_lua) | 每个Nginx worker 进程启动时执行。                            |
| `:certificate()`   | [ssl_certificate_by_lua_block](https://github.com/openresty/lua-nginx-module#ssl_certificate_by_lua_block) | 在SSL握手的SSL证书服务阶段执行。                             |
| `:rewrite()`       | [rewrite_by_lua_block](https://github.com/openresty/lua-nginx-module#rewrite_by_lua_block) | 在作为重写阶段处理程序从客户端接收时针对每个请求执行。在这个阶段，注意到`api`和`consumer`都没有被识别出来，因此，如果插件被配置为全局插件，这个处理程序将被执行！ |
| `:access()`        | [access_by_lua](https://github.com/openresty/lua-nginx-module#access_by_lua) | 针对客户端的每个请求执行，并在代理上游服务之前执行。         |
| `:header_filter()` | [header_filter_by_lua](https://github.com/openresty/lua-nginx-module#header_filter_by_lua) | 从上游服务接收到所有响应头字节时执行。                       |
| `:body_filter()`   | [body_filter_by_lua](https://github.com/openresty/lua-nginx-module#body_filter_by_lua) | 从上游服务接收到的响应体的每个块执行。由于响应被流回到客户端，所以它可以超过缓冲区大小，并且通过块被流传输块。因此如果响应大，可以多次调用该方法。有关更多详细信息，请参阅`lua-nginx-module`文档。 |
| `:log()`           | [log_by_lua](https://github.com/openresty/lua-nginx-module#log_by_lua) | 最后一个响应字节发送到客户端时被执行。                       |

所有这些功能都采用Kong给出的一个参数：插件的配置。此参数是一个简单的Lua表，并将包含您的用户定义的值，根据您选择的模式。更多的在下一章。

------

### handler.lua规范

`handler.lua`文件必须返回一个实现要执行的函数的表。为了简洁起见，这里是一个注释的示例模块，实现所有可用的方法：

注意：Kong使用[rxi / classic](https://github.com/rxi/classic)模块来模拟Lua中的类，并简化了继承模式。

```
-- Extending the Base Plugin handler is optional, as there is no real
-- concept of interface in Lua, but the Base Plugin handler's methods
-- can be called from your child implementation and will print logs
-- in your `error.log` file (where all logs are printed).
local BasePlugin = require "kong.plugins.base_plugin"
local CustomHandler = BasePlugin:extend()

-- Your plugin handler's constructor. If you are extending the
-- Base Plugin handler, it's only role is to instanciate itself
-- with a name. The name is your plugin name as it will be printed in the logs.
function CustomHandler:new()
  CustomHandler.super.new(self, "my-custom-plugin")
end

function CustomHandler:init_worker(config)
  -- Eventually, execute the parent implementation
  -- (will log that your plugin is entering this context)
  CustomHandler.super.init_worker(self)

  -- Implement any custom logic here
end

function CustomHandler:certificate(config)
  -- Eventually, execute the parent implementation
  -- (will log that your plugin is entering this context)
  CustomHandler.super.certificate(self)

  -- Implement any custom logic here
end

function CustomHandler:rewrite(config)
  -- Eventually, execute the parent implementation
  -- (will log that your plugin is entering this context)
  CustomHandler.super.rewrite(self)

  -- Implement any custom logic here
end

function CustomHandler:access(config)
  -- Eventually, execute the parent implementation
  -- (will log that your plugin is entering this context)
  CustomHandler.super.access(self)

  -- Implement any custom logic here
end

function CustomHandler:header_filter(config)
  -- Eventually, execute the parent implementation
  -- (will log that your plugin is entering this context)
  CustomHandler.super.header_filter(self)

  -- Implement any custom logic here
end

function CustomHandler:body_filter(config)
  -- Eventually, execute the parent implementation
  -- (will log that your plugin is entering this context)
  CustomHandler.super.body_filter(self)

  -- Implement any custom logic here
end

function CustomHandler:log(config)
  -- Eventually, execute the parent implementation
  -- (will log that your plugin is entering this context)
  CustomHandler.super.log(self)

  -- Implement any custom logic here
end

-- This module needs to return the created table, so that Kong
-- can execute those functions.
return CustomHandler
```

当然，您的插件本身的逻辑可以在另一个模块中抽象出来，并从您的`handler`模块中调用。许多现有的插件在逻辑冗余时已经选择了这种模式，但它完全是可选的：

```
local BasePlugin = require "kong.plugins.base_plugin"

-- The actual logic is implemented in those modules
local access = require "kong.plugins.my-custom-plugin.access"
local body_filter = require "kong.plugins.my-custom-plugin.body_filter"

local CustomHandler = BasePlugin:extend()

function CustomHandler:new()
  CustomHandler.super.new(self, "my-custom-plugin")
end

function CustomHandler:access(config)
  CustomHandler.super.access(self)

  -- Execute any function from the module loaded in `access`,
  -- for example, `execute()` and passing it the plugin's configuration.
  access.execute(config)
end

function CustomHandler:body_filter(config)
  CustomHandler.super.body_filter(self)

  -- Execute any function from the module loaded in `body_filter`,
  -- for example, `execute()` and passing it the plugin's configuration.
  body_filter.execute(config)
end

return CustomHandler
```

------

### 插件执行顺序

```
注意：这仍然是一个正在进行中的API。关于未来可以配置插件执行顺序的想法，请参阅Mashape / kong＃267。
```

一些插件可能取决于其他人执行某些操作的执行。例如，依赖于消费者身份的插件必须在验证插件之后运行。考虑到这一点，Kong定义了插件执行之间的优先级，以确保执行顺序能够得到遵守。

您的插件的优先级可以通过在返回的处理程序表中接受一个数字的属性进行配置：

```
CustomHandler.PRIORITY = 10
```

优先级越高，您的插件相对于其他插件的阶段（例如`:access()`，`:log()`等）被执行的时间越早。当前的身份验证插件的优先级为1000。

## 插件开发 - 存储配置

### 模块

```
"kong.plugins.<plugin_name>.schema"
```

------

大多数情况下，您的插件可以配置为满足用户的所有需求。您的插件的配置存储在Kong的数据存储区中，以检索它，并在插件执行时将其传递给您的handler.lua方法。

配置由Kong中的Lua表组成，我们称之为`schema`。它包含通过Admin API启用插件时用户将设置的键/值属性。Kong为您提供验证插件的用户配置的方法。

当用户向Admin API发出请求以在给定的API and/or Consumer上启用或更新插件时，您的插件的配置正在针对您的模式进行验证。

例如，用户执行以下请求：

```
$ curl -X POST http://kong:8001/apis/<api name>/plugins \
    -d "name=my-custom-plugin" \
    -d "config.foo=bar"
```

如果配置对象的所有属性根据您的架构有效，那么API将返回`201 Created`，并且该插件将与其配置（`{foo =“bar”}`）一起存储在数据库中。如果配置无效，Admin API将返回`400 Bad Request`和相应的错误消息。

------

### schema.lua规范

此模块将返回一个具有属性的Lua表，该属性将定义用户以后可以配置插件。可用属性有：

| 属性名称      | LUA类型  | 默认值  | 描述                                                         |
| ------------- | -------- | ------- | ------------------------------------------------------------ |
| `no_consumer` | Boolean  | `false` | 如果为真，则无法将此插件应用于特定的消费者。此插件必须仅适用于API范围。例如：认证插件。 |
| `fields`      | Table    | `{}`    | 你的插件的架构。一个可用属性及其规则的键/值表                |
| `self_check`  | Function | `nil`   | 如果要在接受插件的配置之前执行任何自定义验证，则实现该功能。 |

`self_check`功能必须如下实现：

```
-- @param `schema` A table describing the schema (rules) of your plugin configuration.
-- @param `config` A key/value table of the current plugin's configuration.
-- @param `dao` An instance of the DAO (see DAO chapter).
-- @param `is_updating` A boolean indicating wether or not this check is performed in the context of an update.
-- @return `valid` A boolean indicating if the plugin's configuration is valid or not.
-- @return `error` A DAO error (see DAO chapter)
```

以下是一个潜在的`schema.lua`文件示例：

```
return {
  no_consumer = true, -- this plugin will only be API-wide,
  fields = {
    -- Describe your plugin's configuration's schema here.
  },
  self_check = function(schema, plugin_t, dao, is_updating)
    -- perform any custom verification
    return true
  end
}
```

### 描述你的配置概要

`schema.lua`文件的`fields`属性描述了插件配置的模式。它是一个灵活的键/值表，其中每个键将是您的插件的有效配置属性，并且每个值都是描述该属性的规则的表。例如：

```
fields = {
    some_string = {type = "string", required = true},
    some_boolean = {type = "boolean", default = false},
    some_array = {type = "array", enum = {"GET", "POST", "PUT", "DELETE"}}
  }
```

以下是属性的接受规则列表：

| 规则        | LUA类型  | 允许的值                                                     | 描述                                                         |
| ----------- | -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `type`      | string   | "id", "number", "boolean", "string", "table", "array", "url", "timestamp" | 验证属性的类型。                                             |
| `required`  | boolean  |                                                              | 默认值：false。如果为true，则属性必须存在于配置中。          |
| `unique`    | boolean  |                                                              | 默认值：false。如果为true，则该值必须是唯一的（见下面的注释）。 |
| `default`   | any      |                                                              | 如果配置中未指定该属性，则将该属性设置为给定值。             |
| `immutable` | boolean  |                                                              | 默认值：false。如果为true，则在创建插件配置后，不允许更新该属性。 |
| `enum`      | table    | 整数索引表                                                   | 属性的接受值列表。此列表中未包含的任何值将不被接受。         |
| `regex`     | string   | 有效的PCRE正则表达式                                         | 一个用于验证该属性值的正则表达式。                           |
| `schema`    | table    | 嵌套模式定义                                                 | 如果属性的类型是表，则定义要验证这些子属性的模式。           |
| `func`      | function |                                                              | 对属性执行任何自定义验证的功能。有关其参数和返回值，请参阅后面的示例。 |

- *type*

  : 将转换从请求参数中检索的值。如果该类型不是本地Lua类型之一，那么会对其执行自定义验证：

  - **id**: 必须是一个字符串
  - **timestamp**: 必须是数字
  - **url**: 必须是有效的网址
  - **array**: 必须是一个整数索引表（相当于Lua中的数组）。在Admin API中，这样的数组可以通过在请求的正文中具有不同值的属性的键多次发送，或者通过单个主体参数以逗号分隔。

- **unique**: 该属性对于插件配置没有意义，但是当插件需要在数据存储中存储自定义实体时使用该属性。

- **schema**: 如果需要对嵌套属性进行深化验证，则此字段允许您创建嵌套模式。模式验证是递归的。任何级别的嵌套都是有效的，但请注意，这将影响插件的可用性。

- 附加到配置对象但不存在于schema中的任何属性也将使所述配置无效。

------

#### 举例：

`key-auth`插件的`schema.lua`文件定义了API密钥的接受参数名称的默认列表，以及默认设置为false的布尔值：

```
-- schema.lua
return {
  no_consumer = true,
  fields = {
    key_names = {type = "array", required = true, default = {"apikey"}},
    hide_credentials = {type = "boolean", default = false}
  }
}
```

因此，当在`handler.lua`中实现插件的`access()`函数时，并且给予用户启用了默认值的插件，您可以访问：

```
-- handler.lua
local BasePlugin = require "kong.plugins.base_plugin"
local CustomHandler = BasePlugin:extend()

function CustomHandler:new()
  CustomHandler.super.new(self, "my-custom-plugin")
end

function CustomHandler:access(config)
  CustomHandler.super.access(self)

  print(config.key_names) -- {"apikey"}
  print(config.hide_credentials) -- false
end

return CustomHandler
```

------

一个更复杂的例子，可用于最终的日志插件：

```
-- schema.lua

local function server_port(given_value, given_config)
  -- Custom validation
  if given_value > 65534 then
    return false, "port value too high"
  end

  -- If environment is "development", 8080 will be the default port
  if given_config.environment == "development" then
    return true, nil, {port = 8080}
  end
end

return {
  fields = {
    environment = {type = "string", required = true, enum = {"production", "development"}}
    server = {
      type = "table",
      schema = {
        host = {type = "url", default = "http://example.com"},
        port = {type = "number", func = server_port, default = 80}
      }
    }
  }
}
```

这样的配置将允许用户将配置发布到您的插件，如下所示：

```
$ curl -X POST http://kong:8001/apis/<api name>/plugins \
    -d "name=<my-custom-plugin>" \
    -d "config.environment=development" \
    -d "config.server.host=http://localhost"
```

以下将在handler.lua中可用：

```
-- handler.lua
local BasePlugin = require "kong.plugins.base_plugin"
local CustomHandler = BasePlugin:extend()

function CustomHandler:new()
  CustomHandler.super.new(self, "my-custom-plugin")
end

function CustomHandler:access(config)
  CustomHandler.super.access(self)

  print(config.environment) -- "development"
  print(config.server.host) -- "http://localhost"
  print(config.server.port) -- 8080
end

return CustomHandler
```

------

## 插件开发 - 访问数据存储

Kong通过我们称之为“DAO”的类与模型层交互。本章将详细介绍与数据存储区进行交互的可用API。

从0.8.0开始，Kong支持两个主要数据存储：Cassandra 3.x.x和PostgreSQL 9.4+。

------

### DAO工厂

Kong的所有实体都以下列方式表示：

- 描述实体在数据存储中涉及哪个表的模式，其字段上的约束，如外键，非空约束等...此模式是插件配置一章中描述的表。
- 映射到当前使用的数据库（Cassandra或PostgreSQL）的`DAO`类的实例。该类的方法使用模式并公开方法来插入，更新，查找和删除该类型的实体。

Kong的核心实体是：Apis,Consumers and Plugins。这些实体中的每一个都可以通过其相应的DAO实例进行交互，通过DAO Factory实例可以实现。DAO工厂负责加载这些核心实体的DAO以及任何其他实体，例如通过插件提供。

DAO工厂是Kong的singleton instance(单例接口)，因此可以通过`singletons`模块访问：

```
local singletons = require "kong.singletons"

-- Core DAOs
local apis_dao = singletons.dao.apis
local consumers_dao = singletons.dao.consumers
local plugins_dao = singletons.dao.plugins
```

------

### The DAO Lua API

DAO类负责在数据存储区中的给定表上执行的操作，通常映射到Kong中的实体。所有底层支持的数据库（目前为Cassandra和PostgreSQL）都遵循相同的接口，从而使DAO与所有数据库兼容。

例如，插入一个API就像：

```
local singletons = require "kong.singletons"
local dao = singletons.dao

local inserted_api, err = dao.apis:insert({
  name = "mockbin",
  hosts = { "mockbin.com" },
  upstream_url = "http://mockbin.com"
})
```

------

## 插件开发 - 自定义实体

### 模块

```
"kong.plugins.<plugin_name>.schema.migrations"
"kong.plugins.<plugin_name>.daos"
```

------

您的插件可能需要存储比在数据库中存储的配置更多的内容。在这种情况下，Kong可以在主数据存储之上提供抽象，可以让您存储自定义实体。

如前一章所述，Kong通过我们称之为“DAO”的classes与模型层相互作用，通常被称为“DAO工厂”。本章将介绍如何为您的实体提供抽象。

------

### 创建一个migration文件

一旦定义了您的模型，您必须创建迁移模块，这些模块将由Kong执行，以创建您的实体的记录将被存储在其中的表。迁移文件简单地保存迁移数组，并返回它们。

由于Kong `0.8.0`，Cassandra和PostgreSQL都支持，这要求您的插件实现其两个数据库的迁移。

每个迁移必须具有唯一的名称，以及`up`,`down`字段。这样的字段可以是用于简单迁移的SQL / CQL查询的字符串，也可以是复杂的Lua代码。当Kong向前移动时，`up`字段将被执行。它必须使您的数据库的架构达到插件所需的最新状态。 `down`字段必须执行必要的操作才能将模式恢复到之前的状态，在运行`up`之前。

这种方法的主要优点之一是，如果您需要发布修改模型的新版本的插件，则可以在发布插件之前，将新的迁移添加到数组中。另一个好处是还可以恢复这种迁移。

如文件结构章节所述，迁移模块必须命名为：

```
"kong.plugins.<plugin_name>.migrations.cassandra"
"kong.plugins.<plugin_name>.migrations.postgres"
```

以下是如何定义迁移文件以存储API密钥的示例：

```
-- cassandra.lua
return {
  {
    name = "2015-07-31-172400_init_keyauth",
    up =  [[
      CREATE TABLE IF NOT EXISTS keyauth_credentials(
        id uuid,
        consumer_id uuid,
        key text,
        created_at timestamp,
        PRIMARY KEY (id)
      );

      CREATE INDEX IF NOT EXISTS ON keyauth_credentials(key);
      CREATE INDEX IF NOT EXISTS keyauth_consumer_id ON keyauth_credentials(consumer_id);
    ]],
    down = [[
      DROP TABLE keyauth_credentials;
    ]]
  }
}
-- postgres.lua
return {
  {
    name = "2015-07-31-172400_init_keyauth",
    up = [[
      CREATE TABLE IF NOT EXISTS keyauth_credentials(
        id uuid,
        consumer_id uuid REFERENCES consumers (id) ON DELETE CASCADE,
        key text UNIQUE,
        created_at timestamp without time zone default (CURRENT_TIMESTAMP(0) at time zone 'utc'),
        PRIMARY KEY (id)
      );

      DO $$
      BEGIN
        IF (SELECT to_regclass('public.keyauth_key_idx')) IS NULL THEN
          CREATE INDEX keyauth_key_idx ON keyauth_credentials(key);
        END IF;
        IF (SELECT to_regclass('public.keyauth_consumer_idx')) IS NULL THEN
          CREATE INDEX keyauth_consumer_idx ON keyauth_credentials(consumer_id);
        END IF;
      END$$;
    ]],
    down = [[
      DROP TABLE keyauth_credentials;
    ]]
  }
}
```

- name: 必须是唯一的字符串。格式并不重要，但可以帮助您在开发插件时调试问题，因此请务必以相关方式命名。
- up: 当kong迁移时执行。
- down: 当kong回滚时执行。

虽然Postgres，Cassandra不支持“NOT NULL”，“UNIQUE”或“FOREIGN KEY”等约束，但是在定义模型的架构时，Kong可以提供这些功能。请记住，对于PostgreSQL和Cassandra，此模式将是相同的，因此，您可能会为与Cassandra一起使用的纯SQL模式进行权衡。

**重要信息：如果您的架构使用唯一的约束，那么对于Cassandra，Kong将强制执行，但对于Postgres，您必须在migrations文件中设置此约束。**

### 从DAO工厂检索您的定制DAO

要使DAO工厂加载您的自定义DAO，您只需要定义实体的模式（就像描述插件配置的模式）。此模式包含更多值，因为它必须描述实体在数据存储中涉及的表，其字段上的约束，如外键，非空约束等。

该schema将在名为：

```
"kong.plugins.<plugin_name>.daos"
```

一旦该模块返回您的实体模式，并假设您的插件由Kong加载（请参阅`kong.yml`中的`custom_plugins`属性），DAO工厂将使用它来实现DAO对象。

以下是一个示例，说明如何定义模式以将API密钥存储在他或她的数据库中：

```
-- daos.lua
local SCHEMA = {
  primary_key = {"id"},
  table = "keyauth_credentials", -- the actual table in the database
  fields = {
    id = {type = "id", dao_insert_value = true}, -- a value to be inserted by the DAO itself (think of serial ID and the uniqueness of such required here)
    created_at = {type = "timestamp", immutable = true, dao_insert_value = true}, -- also interted by the DAO itself
    consumer_id = {type = "id", required = true, foreign = "consumers:id"}, -- a foreign key to a Consumer's id
    key = {type = "string", required = false, unique = true} -- a unique API key
  }
}

return {keyauth_credentials = SCHEMA} -- this plugin only results in one custom DAO, named `keyauth_credentials`
```

由于您的插件可能需要处理多个自定义DAO（在要存储多个实体的情况下），该模块必须返回一个键/值表，其中键是DAO工厂中自定义DAO可用的名称。

您将在模式定义中注意到一些新属性（与schema.lua文件相比）：

| 属性名称                    | LUA类型    | 描述                                                         |
| --------------------------- | ---------- | ------------------------------------------------------------ |
| `primary_key`               | 整数索引表 | 您列系列主键的每个部分的一个数组。它还支持复合密钥，即使所有Kong实体当前使用简单的`id`来管理API的可用性。如果你的主键是复合的，那么只包括您的分区键。 |
| `fields.*.dao_insert_value` | Boolean    | 如果为true，则指定此字段将由DAO自动填充（在base_dao实现中），具体取决于其类型。类型`id`的属性将是生成的uuid，并且`timestamp`具有第二精度的时间戳。 |
| `fields.*.queryable`        | Boolean    | 如果为true，则指定Cassandra在指定列上维护索引。这允许查询此列过滤的列集合字段。 |
| `fields.*.foreign`          | String     | 指定此列是另一个实体的列的外键。格式为：`dao_name:column_name`。这使得Cassandra不支持外键。当父行将被删除时，Kong还将删除包含父列的列值的行。 |

您的DAO现在将由DAO工厂加载，并作为其属性之一提供：

```
local singletons = require "kong.singletons"
local dao_factory = singletons.dao

local keys_dao = dao_factory.keyauth_credentials

local key_credential, err = keys_dao:insert({
  consumer_id = consumer.id,
  key = "abcd"
}
```

可以从DAO工厂访问的DAO名称（`keyauth_credentials`）取决于在`daos.lua`的返回表中导出DAO的键。

------

### 缓存自定义实体

有时每个请求/响应都需要自定义实体，这反过来又会触发每次数据存储上的查询。这是非常低效的，因为查询数据存储会增加延迟并降低请求/响应速度，并导致数据存储区上的负载增加可能会影响数据存储的性能本身，也可能影响其他Kong节点。

当每个请求/响应都需要一个自定义实体时，通过利用Kong提供的内存中缓存API来缓存内存是个好习惯。

下一章将专注于缓存自定义实体，并在数据存储区中更改时使其无效：缓存自定义实体。

------

## 插件开发 - 缓存自定义实体

### 模块

```
"kong.plugins.<plugin_name>.daos"
"kong.plugins.<plugin_name>.hooks"
```

------

您的插件可能需要经常访问每个请求和/或响应的自定义实体（在上一章中介绍）。通常加载它们一次，并将它们缓存在内存中，可以显着提高性能，同时确保数据存储不受负载增加的压力。

想想一个需要在每个请求上验证api密钥的api密钥验证插件，从而在每个请求上从数据存储区加载自定义凭证对象。当客户端与请求一起提供api密钥时，通常会查询数据存储区以检查该密钥是否存在，然后阻止请求或检索用户ID以标识用户。这将在每个请求上发生，这将是非常低效的：

- 查询数据存储区会增加每个请求的延迟，从而使请求处理速度更慢。
- 数据存储还将受到负载增加的影响，可能会导致数据存储崩溃或减慢，这又会影响每个Kong节点。

为了避免每次查询数据存储区，我们都可以在节点上缓存自定义实体内存，以便频繁的实体查找不会每次（仅第一次）触发数据存储查询，但是在内存中发生，从数据存储区（特别是在重负载情况下）查询更快，更可靠。

```
注意：当缓存内存中的自定义实体时，您还需要提供一个无效机制，在“hooks.lua”文件中实现。
```

------

### 缓存自定义实体

一旦您定义了自定义实体，就可以通过要求`database_cache`依赖关系在代码中缓存内存：

```
local cache = require "kong.tools.database_cache"
```

有两个级别的缓存：

1. Lua内存缓存（本地到nginx worker）这可以保存任何类型的Lua值。
2. 共享内存缓存 - SHM（本地到nginx节点，但在所有workers之间共享）这只能保存标量值，因此需要（反）序列化。

当从数据库中获取数据时，它将被存储在两个缓存中。现在如果同一个工作进程再次请求数据，它将从Lua内存缓存中检索先前反序列化的数据。如果同一个nginx节点内的一个不同的worker请求该数据，它会在SHM中找到数据，并将其反序列化（并将其存储在自己的Lua内存缓存中），然后返回。

该模块公开了以下功能：

| 函数名                                                   | 描述                                                         |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| `ok, err = cache.set(key, value, ttl)`                   | 使用指定的键将Lua对象存储到内存中缓存（可选ttl以秒为单位）。该值可以是任何Lua类型，包括表。返回true或false，如果操作失败，则返回err。 |
| `value = cache.get(key)`                                 | 检索存储在特定键中的Lua对象。                                |
| `cache.delete(key)`                                      | 删除存储在指定键的缓存对象。                                 |
| `ok, err = cache.sh_add(key, value, ttl)`                | 将新值添加到SHM缓存中，可选ttl（以秒为单位）（如果nil不会过期） |
| `ok, err = cache.sh_set(key, value, ttl)`                | 在SHM中的指定键下设置一个新值，可选ttl（以秒为单位）（如果nil不会过期） |
| `value = cache.sh_get(key)`                              | 返回存储在SHM下的值，如果没有找到，则返回nil                 |
| `cache.sh_delete(key)`                                   | 从SHMs删除指定key的值                                        |
| `newvalue, err = cache.sh_incr(key, amount)`             | 在指定的键下增加存储在SHM中的数量，以指定的单位数量。该数字需要已经存在于缓存中，否则将返回错误。如果成功，则返回新增值，否则返回错误。 |
| `value, ... = cache.get_or_set(key, ttl, function, ...)` | 这是一个使用指定键检索对象的实用方法，但如果对象为nil，则将执行传递的函数，而返回值将用于将对象存储在指定的键。这有效地确保该对象仅从数据存储区一次加载，因为每次其他调用将从内存中缓存加载对象。 |

返回我们的认证插件示例，要使用特定的api密钥查找凭据，我们将会写下如下：

```
-- access.lua

local function load_entity_key(api_key)
  -- IMPORTANT: the callback is executed inside a lock, hence we cannot terminate
  -- a request here, we MUST always return.
  local apikeys, err = dao.apikeys:find_by_keys({key = api_key}) -- Lookup in the datastore
  if err then
    return nil, err     -- errors must be returned, not dealt with here
  end
  if not apikeys then
    return nil          -- nothing was found
  end
  -- assuming the key was unique, we always only have 1 value...
  return apikeys[1] -- Return the credential (this will be also stored in-memory)
end


local credential
-- Retrieve the apikey from the request querystring
local apikey = request.get_uri_args().apikey
if apikey then -- If the apikey has been passed, we can check if it exists

  -- We are using cache.get_or_set to first check if the apikey has been already stored
  -- into the in-memory cache at the key: "apikeys."..apikey
  -- If it's not, then we lookup the datastore and return the credential object. Internally
  -- cache.get_or_set will save the value in-memory, and then return the credential.
  credential, err = cache.get_or_set("apikeys."..apikey, nil, load_entity_key, apikey)
  if err then
    -- here we can deal with the error returned by the callback
    return response.HTTP_INTERNAL_SERVER_ERROR(err) 
  end
end

if not credential then -- If the credential couldn't be found, show an error message
  return responses.send_HTTP_FORBIDDEN("Invalid authentication credentials")
end
```

通过这样做，不用担心客户端使用该特定api密钥发送多少请求，在第一个请求之后，每次查找将在内存中完成，而不查询数据存储区。

#### 更新或删除自定义实体

每次在数据存储上更新或删除缓存的自定义实体时，例如使用Admin API，它会在数据存储区中的数据与缓存在内存中的数据存储区之间产生矛盾。为了避免这种不一致，我们需要从内存存储器中删除缓存的实体，并强制要求从数据存储区再次请求。为了这样做，我们必须实现一个无效的钩子。

------

### 使自定义实体无效

每当在数据存储区中创建/更新/删除实体时，Kong会通知所有节点上的数据存储区操作，告知执行了哪个命令以及哪个实体受到影响。这发生在APIs, Plugins 和 Consumers，也适用于自定义实体。

由于这种行为，我们可以通过适当的操作来监听这些事件和响应，以便在数据存储区中修改缓存的实体时，我们可以将其从缓存中显式删除，以避免数据存储和缓存本身之间的状态不一致。从内存的缓存中删除它将触发系统再次查询数据存储区，并重新缓存实体。

kong传播的事件是：

| 事件名称         | 描述                 |
| ---------------- | -------------------- |
| `ENTITY_CREATED` | 当任何实体被创建时。 |
| `ENTITY_UPDATED` | 任何实体正在更新时。 |
| `ENTITY_DELETED` | 任何实体被删除时。   |

为了侦听这些事件，我们需要实现`hooks.lua`文件并使用我们的插件进行分发，例如：

```
-- hooks.lua

local events = require "kong.core.events"
local cache = require "kong.tools.database_cache"

local function invalidate_on_update(message_t)
  if message_t.collection == "apikeys" then
    cache.delete("apikeys."..message_t.old_entity.apikey)
  end
end

local function invalidate_on_create(message_t)
  if message_t.collection == "apikeys" then
    cache.delete("apikeys."..message_t.entity.apikey)
  end
end

return {
  [events.TYPES.ENTITY_UPDATED] = function(message_t)
    invalidate_on_update(message_t)
  end,
  [events.TYPES.ENTITY_DELETED] = function(message_t)
    invalidate_on_create(message_t)
  end
}
```

在上面的示例中，插件正在侦听`ENTITY_UPDATED`和`ENTITY_DELETED`事件，并通过调用适当的函数进行响应。 `message_t`表包含事件属性：

| 属性名称     | 类型   | 描述                                 |
| ------------ | ------ | ------------------------------------ |
| `collection` | String | 资料储存库中的集合受到操作的影响。   |
| `entity`     | Table  | 最近更新的实体，或删除或创建的实体。 |
| `old_entity` | Table  | 仅适用于更新事件，旧版本的实体。     |

在`entity`和`old_entity`属性中传输的实体不具有在模式中定义的所有字段，而只包含一个子集。这是必需的，因为每个事件都是以一个有效载荷大小限制为512字节的UDP数据包发送的。该子集由模式中的`marshall_event`函数返回，您可以选择实现。

#### marshall_event

此函数将自定义实体序列化到最小版本，仅包含稍后需要在`hooks.lua`中使用的字段。如果`marshall_event`未被实现，默认情况下，Kong不发送任何实体字段值以及事件。

例如：

```
-- daos.lua

local SCHEMA = {
  primary_key = {"id"},
  -- clustering_key = {}, -- none for this entity
  fields = {
    id = {type = "id", dao_insert_value = true},
    created_at = {type = "timestamp", dao_insert_value = true},
    consumer_id = {type = "id", required = true, queryable = true, foreign = "consumers:id"},
    apikey = {type = "string", required = false, unique = true, queryable = true}
  },
  marshall_event = function(self, t) -- This is related to the invalidation hook
    return { id = t.id, consumer_id = t.consumer_id, apikey = t.apikey }
  end
}
```

在上面的示例中，自定义实体提供了一个`marshall_event`函数，它返回一个具有`id`，`consumer_id`和`apikey`字段的对象。在我们的钩子中，我们不需要`creation_date`来使实体无效，所以我们不在乎在事件中传播它。参数中的`t`表是其所有字段的原始对象。

```
注意：正在返回的Lua表的JSON序列化不能超过512个字节，以便将整个事件放于一个UDP数据包。不符合这种约束将会阻止无效宣传事件的传播，从而造成节点间的数据不一致。
```

------

### 扩展Admin API

您可能知道，Admin API是Kong用户与Kong进行沟通以设置其API和插件的地方。它们可能还需要与您为插件实现的自定义实体进行交互（例如，创建和删除API密钥）。您将要做的事情是扩展Admin API，我们将在下一章中详细介绍：扩展Admin API。

------

## 插件开发 - 扩展Admin API

### 模块

```
"kong.plugins.<plugin_name>.api"
```

------

Admin API是用户配置Kong的接口。如果您的插件具有自定义实体或管理要求，则需要扩展Admin API。这允许您公开您自己的端点并实现您自己的管理逻辑。其中一个典型的例子是API密钥的创建，检索和删除（通常称为“CRUD操作”）。

Admin API是一个[Lapis](http://leafo.net/lapis/)应用程序，Kong的抽象级别使您可以轻松添加端点。

```
注意：本章假定您具有Lapis的相关知识。
```

------

### 将端点添加到Admin API

如果endpoinds按照如下的模块中定义，Kong将检测并加载endpoints。

```
"kong.plugins.<plugin_name>.api"
```

该模块必须返回一个包含描述路由的字符串的表（请参阅Lapis路由和URL模式）和它们支持的HTTP动词。然后路由被分配一个简单的处理函数。

然后将此表提供给Lapis（请参阅[Lapis处理HTTP动词文档](http://leafo.net/lapis/reference/actions.html#handling-http-verbs)）。例：

```
return {
  ["/my-plugin/new/get/endpoint"] = {
    GET = function(self, dao_factory, helpers)
      -- ...
    end
  }
}
```

处理函数有三个参数，它们按顺序排列：

- `self`: 请求对象。请参阅[Lapis请求对象](http://leafo.net/lapis/reference/actions.html#request-object)
- `dao_factory`: DAO工厂。请参阅本指南的数据存储区一章。
- `helpers`: 一个包含几个助手的表，如下所述。

除了支持的HTTPS动词之外，路由表还可以包含两个其他密钥：

- before: 在Lapis中，在执行的动词动作之前运行before_filter。
- on_error: 一个自定义的错误处理函数，它覆盖了由Kong提供的函数。请参阅Lapis的[捕获可恢复错误](http://leafo.net/lapis/reference/exception_handling.html#capturing-recoverable-errors)文档。

------

### Helpers

在Admin API处理请求时，有时您希望发回响应并处理错误，以帮助您这样做，第三个参数`helpers`是具有以下属性的表：

- `responses`: 具有帮助函数的模块发送HTTP响应。
- `yield_error`: 来自Lapis的[yield_error](http://leafo.net/lapis/reference/exception_handling.html#capturing-recoverable-errors)函数。当你的处理程序遇到错误（例如从DAO）时调用。由于所有Kong错误都是具有上下文的表，因此可以根据错误（内部服务器错误，错误请求等）发送适当的响应代码。

#### crud_helpers

由于您将在您的端点执行的大多数操作将是CRUD操作，您还可以使用`kong.api.crud_helpers`模块。此模块为您提供任何插入，搜索，更新或删除操作的帮助程序，并执行必要的DAO操作并使用适当的HTTP状态代码进行回复。它还为您提供了从路径中检索参数的功能，例如API的名称或ID，或Consumer的用户名或ID。

举例：

```
local crud = require "kong.api.crud_helpers"

return {
  ["/consumers/:username_or_id/key-auth/"] = {
    before = function(self, dao_factory, helpers)
      crud.find_consumer_by_username_or_id(self, dao_factory, helpers)
      self.params.consumer_id = self.consumer.id
    end,

    GET = function(self, dao_factory, helpers)
      crud.paginated_set(self, dao_factory.keyauth_credentials)
    end,

    PUT = function(self, dao_factory)
      crud.put(self.params, dao_factory.keyauth_credentials)
    end,

    POST = function(self, dao_factory)
      crud.post(self.params, dao_factory.keyauth_credentials)
    end
  }
}
```

------

## 插件开发 - 编写测试

如果你对你的插件负责，你可能想为它编写测试。单元测试Lua很简单，并且有许多测试框架可用。但是，您也可能需要编写集成测试。再次，kong能够给你提供支援。

------

### 编写集成测试

Kong的首选测试框架[busted](http://olivinelabs.com/busted/)通过[resty-cli](https://github.com/openresty/resty-cli)翻译运行，尽管如果愿意，您可以自由使用另一个。在Kong存储库中，可以在`bin / busted`找到`busted`的可执行文件。

Kong在您的测试套件中为您提供了一个帮助程序来启动和停止Lua：`spec.helpers`。此助手还提供了在运行测试之前在数据存储区中插入fixtures的方法，以及删除，以及各种其他helper。

如果您在自己的存储库中编写插件，则需要复制以下文件，直到Kong测试框架被释放：

- `bin/busted`: busted 的可执行文件与resty-cli解释器一起运行
- `spec/helpers.lua`: Kong的helper函数 启动/关闭 busted
- `spec/kong_tests.conf`: 一个用于使用helpers模块运行test Kong实例的配置文件

假设`spec.helpers`模块在您的`LUA_PATH`中可用，您可以使用以下`busted`的Lua代码来启动和停止Kong：

```
local helpers = require "spec.helpers"

describe("my plugin", function()

  local proxy_client
  local admin_client

  setup(function()
    assert(helpers.dao.apis:insert {
      name         = "test-api",
      hosts        = "test.com",
      upstream_url = "http://httpbin.org"
    })

    -- start Kong with your testing Kong configuration (defined in "spec.helpers")
    assert(helpers.start_kong())

    admin_client = helpers.admin_client(timeout?)
  end)

  teardown(function()
    if admin_client then
      admin_client:close()
    end

    helpers.stop_kong()
  end)

  before_each(function()
    proxy_client = helpers.proxy_client(timeout?)
  end)

  after_each(function()
    if proxy_client then
      proxy_client:close()
    end
  end)

  describe("thing", function()
    it("should do thing", function()
      -- send requests through Kong
      local res = assert(proxy_client:send {
        method = "GET",
        path   = "/get",
        headers = {
          ["Host"] = "test.com"
        }
      })

      local body = assert.res_status(200, res)

      -- body is a string containing the response
    end)
  end)
end)
```

**提醒：通过test Kong配置文件，Kong运行在代理监听端口8100和端口8101上的Admin API。**

------

## 插件开发 - （卸载）安装你的插件

Kong的自定义插件由Lua源文件组成，需要位于每个Kong节点的文件系统中。本指南将为您提供有助于使Kong节点了解您的自定义插件的分步说明。

这些步骤应该应用到您的Kong集群中的每个节点，以确保自定义插件在每个节点上都可用。

### 目录

- Packaging sources
- Installing the plugin
- Load the plugin
- Verify loading the plugin
- Removing a plugin
- Distribute your plugin
- Troubleshooting

### Packaging sources

您可以使用常规打包策略（例如`tar`），也可以使用LuaRocks包管理器为您执行。我们建议您使用LuaRocks，因为它与Kong一起使用官方发行包之一。

使用LuaRocks时，您必须创建一个指定软件包内容的`rockspec`文件。有关示例，请参阅[Kong插件模板](https://github.com/luarocks/luarocks/wiki/Creating-a-rock)，有关格式的更多信息，请参阅Rockspecs上的[LuaRocks文档](https://github.com/luarocks/luarocks/wiki/Creating-a-rock)。

使用以下命令（从插件repo）打包你的项目：

```
# install it locally (based on the `.rockspec` in the current directory)
$ luarocks make

# pack the installed rock
$ luarocks pack <plugin-name> <version>
```

假设你的插件rockspec被称为`kong-plugin-myPlugin-0.1.0-1.rockspec`，以上将成为如下;

```
$ luarocks pack kong-plugin-myPlugin 0.1.0-1
```

LuaRocks `pack`命令现在已经创建了一个`.rock`文件（这只是一个zip文件，其中包含安装包所需的一切）。

如果您不使用或不能使用LuaRocks，则使用`tar`将您的插件所在的`.lua`文件打包到`.tar.gz`存档中。如果目标系统上有LuaRocks，还可以包括`.rockspec`文件。

此存档的内容应接近于以下内容：

```
$ tree <plugin-name>
<plugin-name>
├── INSTALL.txt
├── README.md
├── kong
│   └── plugins
│       └── <plugin-name>
│           ├── handler.lua
│           └── schema.lua
└── <plugin-name>-<version>.rockspec
```

------

### Installing the plugin

要使Kong节点能够使用自定义插件，必须在主机的文件系统上安装自定义插件的Lua源。有多种方法：通过LuaRocks，或手动。选择一个，然后跳转到第3节。

1.通过LuaRocks从创建的“rock”安装

`.rock`文件是一个自包含的软件包，可以在本地安装或从远程服务器安装。

如果您的系统中安装了`luarocks`实用程序（如果您使用其中一个官方安装包，则可能会出现此情况），则可以在LuaRocks树（LuaRocks安装Lua模块的目录）中安装“rock”。

可以通过以下方式进行安装：

```
$ luarocks install <rock-filename>
```

文件名可以是本地名称，也可以是任何支持的方法，例如。
`http://myrepository.lan/rocks/myplugin-0.1.0-1.all.rock`

2.通过LuaRocks从源安装

如果您的系统中安装了`luarocks`实用程序（如果您使用其中一个官方安装软件包，则可能是这种情况），则可以在LuaRocks树（LuaRocks安装Lua模块的目录）中安装Lua源。

您可以通过将当前目录更改为提取的存档，其中rockspec文件位于:

```
$ cd <plugin-name>
```

然后运行以下命令：

```
luarocks make
```

这将在系统的LuaRocks树中的`kong / plugins / <plugin-name>`中安装Lua源，其中所有Kong源都已存在。

3.手动安装

安装插件源代码的更加保守的方法是避免“污染”LuaRocks树，而是将Kong指向包含它们的目录。

这通过调整您的Kong配置的`lua_package_path`属性来完成。在引擎盖下，如果您熟悉该属性，则该属性是Lua VM的`LUA_PATH`变量的别名。

这些属性包含用于搜索Lua源的目录的分号分隔列表。您的Kong配置文件应该如此设置：

```
lua_package_path = /<path-to-plugin-location>/?.lua;;
```

where:

```
* `/<path-to-plugin-location>` is the path to the directory containing the
  extracted archive. It should be the location of the `kong` directory
  from the archive.
* `?` is a placeholder that will be replaced by
  `kong.plugins.<plugin-name>` when Kong will try to load your plugin. Do
  not change it.
* `;;` a placeholder for the "the default Lua path". Do not change it.

Example:

The plugin `something` being located on the file system such that the
handler file is:

    /usr/local/custom/kong/plugins/<something>/handler.lua

The location of the `kong` directory is: `/usr/local/custom`, hence the
proper path setup would be:

    lua_package_path = /usr/local/custom/?.lua;;

Multiple plugins:

If you wish to install two or more custom plugins this way, you can set
the variable to something like:

    lua_package_path = /path/to/plugin1/?.lua;/path/to/plugin2/?.lua;;

* `;` is the separator between directories.
* `;;` still means "the default Lua path".

Note: you can also set this property via its environment variable
equivalent: `KONG_LUA_PACKAGE_PATH`.
```

提醒：无论您使用哪种方法来安装插件的源，您仍然必须对Kong群集中的每个节点执行此操作。

------

### Load the plugin

您现在必须将自定义插件的名称添加到Kong配置（每个Kong节点）的custom_plugins列表中：

```
custom_plugins = <plugin-name>
```

如果您使用两个或多个自定义插件，请在其间插入逗号，如下所示：

```
custom_plugins = plugin1,plugin2
```

注意：您还可以通过其环境变量等效设置此属性：`KONG_CUSTOM_PLUGINS`。 提醒：不要忘记更新您的Kong群集中每个节点的`custom_plugins`指令。

------

### Verify loading the plugin

你现在应该可以没有任何问题的开始了。请参阅您的自定义插件的说明，了解如何在API或Consumer对象上启用/配置插件。

要确保您的插件由Kong加载，您可以使用调试日志级别启动Kong：

```
log_level = debug
```

或者:

```
KONG_LOG_LEVEL=debug
```

然后，您将看到正在加载的每个插件的以下日志：

```
[debug] Loading plugin <plugin-name>
```

------

### Removing a plugin

完全删除插件有三个步骤。

1. 从Kong api配置中删除该插件。确保它不再适用于全局或任何API或消费者。对于整个群集，这只能执行一次，不需要重新启动/重新加载。这一步本身将使插件不再使用。但是它仍然可用，并且仍然可以重新应用插件。
2. 通过`custom_plugins`指令（每个Kong节点）删除该插件。在这一步之前请确保步骤1已经执行。在这一步之后，任何人都不可能将插件重新应用于任何Kong API，消费者甚至全局。此步骤需要重新启动/重新加载Kong节点才能生效。
3. 要彻底删除插件，请从每个Kong节点删除与插件相关的文件。在删除文件之前，请务必完成第2步，包括重新启动/重新加载Kong。如果使用LuaRocks安装插件，您可以使用`luarocks remove <plugin-name>`将其删除。

------

### Distribute your plugin

这样做的首选方法是使用Luarocks，一个Lua模块的软件包管理器。它称之为“rocks”模块。您的模块不必住在Kong存储库中，但是如果您想保持您的Kong设置，它将可以存在kong 存储库中。

通过在rockspec文件中定义模块（及其最终依赖关系），您可以通过Luarocks在平台上安装这些模块。您还可以将您的模块上传到Luarocks，并将其提供给所有人！

这里是一个使用“内置”构建类型定义Lua符号及其相应文件中的模块的rockspec示例：

有关示例，请参阅[Kong插件模板](https://github.com/luarocks/luarocks/wiki/Creating-a-rock)，有关格式的更多信息，请参阅Rockspecs上的[LuaRocks文档](https://github.com/luarocks/luarocks/wiki/Creating-a-rock)。

------

### Troubleshooting

由于几个原因，由于配置错误的自定义插件，Kong可能无法启动：

- “plugin is in use but not enabled” -->您从另一个节点配置了一个自定义插件，并且该插件配置位于数据库中，但您尝试启动的当前节点在其`custom_plugins`指令中没有。要解决，请将插件的名称添加到节点的`custom_plugins`指令中。
- "plugin is enabled but not installed" -->该插件的名称存在于`custom_plugins`指令中，但是Kong无法从文件系统加载`handler.lua`源文件。要解决，请确保`lua_package_path`指令正确设置为加载此插件的Lua源。
- "no configuration schema found for plugin" -->该插件已安装，已在`custom_plugins`中启用，但Kong无法从文件系统加载`schema.lua`源文件。要解决，请确保`schema.lua`文件与插件的`handler.lua`文件一起存在。