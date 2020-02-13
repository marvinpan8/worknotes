## kong使用记录

Kong侦听四个端口的请求，默认情况是：

**8000：此端口是Kong用来监听来自客户端的HTTP请求的，并将此请求转发到您的上游服务。这也是本教程中最主要用到的端口。**

8443：此端口是Kong监听HTTP的请求的端口。该端口具有与8000端口类似的行为，但是它只监听HTTPS的请求，**并不会产生转发行为。可以通过配置文件来禁用此端口**。

8001：用于管理员对KONG进行配置的端口。

8444：用于管理员监听HTTPS请求的端口。

在本文中，我们将介绍Kong的路由功能，并详细说明8000端口上的客户端请求如何根据请求头、URI或HTTP被代理到配置中的上游服务。

## 官方插件

目前在Kong的 free plugins中，比较常用的有这么三个：Syslog、File-Log以及Http-Log，下面对这三种插件逐一分析一下。

#### Syslog

顾名思义，这个插件是把Kong中记录的日志给打印到系统日志中，开启插件之后只需要指定需要使用的API，无需做多余的配置，即可在`/var/log/message`中发现对应的日志信息，d 但是系统日志鱼龙混杂，如果需要用到ELK等日志分析工具时，需要做一次数据清洗工作。

#### File-Log

与Syslog一样，File-log的配置也很方便，只需要配置日志路劲就行，开启插件之后，会在对应的对应产生一个logFile。Syslog中提到需要做一些日志清洗工作，但是换成了File-log乍一看好像解决了之前的痛点，实则不然，官方建议这个插件不适合在生产环境中使用，会带来一些性能上的开销，影响正常业务。

#### Http-Log

http-log是我比较推荐的，它的原理是设置一个log-server地址，然后Kong会把日志通过post请求发送到设置的log-server，然后通过log-server把日志给沉淀下来，相比之前两种插件，这一种只要启一个log-server就好了，出于性能考虑，我用Rust实现了一个 [log-server](<https://github.com/Makcy/log-server>)，有兴趣可以参考看一下。

#### prometheus可视化

kong自带的prometheus插件,metrics比较少, 可以网上查一下丰富版的prometheus插件.

比如:https://github.com/yciabaud/kong-plugin-prometheus , 只兼容Kong 0.11版本

现在用这个插件替换kong自带的插件.

最方便的安装方式，一般linux机器上都会自带 luarocks（lua包管理程序），这样一来我们只要把 Plugins 所在的文件夹给移动到服务器的任意目录，然后在该目录下，执行luarocks make 这样一来插件便会自动安装到系统中，不过需要注意的是，此时插件还需要进行手动开启，首先进入/etc/kong/目录，然后cp kong.conf.default kong.conf， 这里注意一定要复制一份单独的kong.conf文件，不能直接对kong.conf.default进行修改，这样是不生效的，然后取消plugin = bundled前面的注释，在这一行后面增加你的插件名，这里注意插件名是不包含前缀 kong-plugin的，重启Kong即可在可视化界面里发现

```properties
plugins = bundled,prometheus
```

在使用新插件之前，需要更新一下数据库：

```bash
bash ./resty.sh kong/bin/kong  migrations up -c kong.conf
```

#### 爬虫控制插件bot-detection

备注：

config.whitelist ：白名单，逗号分隔的正则表达式数组。正则表达式是根据 User-Agent 头部匹配的。
config.blacklist ：黑名单，逗号分隔的正则表达式数组。正则表达式是根据 User-Agent 头部匹配的。

这个字段是用来匹配客户端身份的, 比如是浏览器还是模拟器, 还是python代码.

这个插件已经包含了一个基本的规则列表，这些规则将在每个请求上进行检查。你可以在GitHub上找到这个列表 https://github.com/Kong/kong/blob/master/kong/plugins/bot-detection/rules.lua.  

## 自定义插件

plugins目录增加文件夹，

修改 `kong-1.3.0-0.rockspec`，最后增加加载 lua 的文件

```bash
["kong.plugins.http-anti-replay-attack.handler"] = "kong/plugins/http-anti-replay-attack/handler.lua",
["kong.plugins.http-anti-replay-attack.schema"] = "kong/plugins/http-anti-replay-attack/schema.lua",
["kong.plugins.http-anti-replay-attack.policies"] = "kong/plugins/http-anti-replay-attack/policies/init.lua",
```

修改`constants.lua`中，plugins变量增加

```bash
  "http-anti-replay-attack",
```

vim Dockerfile

```bash
FROM kong:1.3.0-alpine
MAINTAINER pantj <pantj@investoday.com.cn>
COPY http-anti-replay-attack /usr/local/share/lua/5.1/kong/plugins/http-anti-replay-attack
COPY kong-1.3.0-0.rockspec /usr/local/lib/luarocks/rocks-5.1/kong/1.3.0-0
COPY constants.lua /usr/local/share/lua/5.1/kong/
```

## 缓存

链接：https://www.jianshu.com/p/68457b42b84f

作为网关代理层, 缓存一定是必不可少的一环, 在kong的PDK中, 封装了 `lua-resty-mlcache`
 kong的缓存分为两级:

- L1: `Lua memory cache` - 在nginx worker中共享, 可以存储任何Lua值
- L2: `Shared memory cache (SHM)` - 在nginx node的所有worker中共享. 可以存储任何标量值, 但它需要序列化和反序列化, 所以性能会有所下降

> 注: 从数据库中提取数据后，它将同时存储于上述两级缓存中。现在，如果同一个worker进程再次请求数据，它将从Lua内存缓存检索数据。 如果同一个Nginx节点中的不同worker请求该数据，它将在SHM中找到数据，对其进行反序列化（并将其存储在自己的Lua内存缓存中），然后将其返回。

一个典型的使用方式如下:

```lua
-- 通过一个唯一值获取Cache Key
cache_key = singletons.dao.keyauth_credentials:cache_key("api_key")
value, err = singletons.cache:get(cache_key ,nil, load_from_db_func, ....)
---新版本，kong.db kong.cache
local consumer_cache_key = kong.db.consumers:cache_key(conf.anonymous)
local consumer, err = kong.cache:get(consumer_cache_key, nil,
                                                load_consumer_into_memory,
                                                conf.anonymous, true)
```

函数: `value, err = cache:get(key, opts?, cb, ...)` , 如果cache没有值(miss), 则会调用函数`cb`, `cb` 必须返回一个返回值, 可返回需要缓存的值或者`nil`, 需要注意的是返回值为`nil`时, get函数依旧会执行缓存, 但我们可以通过`cache:get()`的第二个参数控制缓存的TTL和negative TTL. 如下代码即代表有数据时, 我们缓存600s, 没有数据时, 缓存`nil`结果40s

```lua
value, err = singletons.cache:get(cache_key ,{
            ttl = 600, --如果有数据, 则缓存的时间, 单位:秒
            neg_ttl = 40 -- 如果没有数据, 单位:秒
        }, load_from_db_func, ....)
```

通过上述API开发者可轻松实现缓存懒加载功能

---

```bash
ngx.var.uri=/dataapi/consensus/est_bsc
kong.request.get_path_with_query()=/dataapi/consensus/est_bsc?begin_date=20180101&end_date=20180131&ind_id=0&sec_cd=000001&fields=&oper_type=0&page=1&page_count=1000&rpt_yr=
```

---

### http-log / kafka-log日志的坑

- 必须配置`All consumers`，而不指定 consumer id,才能打印 401，403之类的返回日志。且401没有consumer属性值。