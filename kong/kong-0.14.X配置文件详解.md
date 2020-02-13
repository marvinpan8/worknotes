

## 配置加载

Kong的默认配置在 `/etc/kong/kong.conf.default` 。如果你通过一个官方的安装包来安装Kong。您可以复制下面的文件，开始配置Kong：

```shell
$ cp /etc/kong/kong.conf.default /etc/kong/kong.conf
```

如果你的配置中的所有值都被考虑，那么Kong将使用默认配置运行。在开始时，Kong可能会查找的几个缺省配置文件位置如下：

```properties
/etc/kong/kong.conf
/etc/kong.conf
```

您可以通过在CLI中使用`-c / -conf`参数自定义配置文件路径来覆盖此默认配置：

```shell
$ kong start --conf /path/to/kong.conf
```

配置格式很简单：注释由字符 `#` 定义。布尔值可以被指定为 `on/off` 或者`true/false`。

## 验证您的配置

您可以使用`check`命令来验证设置的完整性：

```shell
$ kong check <path/to/kong.conf>
configuration at <path/to/kong.conf> is valid
```

这个命令，将检测您当前设置的环境变量，并且在您的设置错误时报错。

此外，您还可以在调试模式下使用CLI，以便更深入地了解正在启动的属性：

```shell
$ kong start -c <kong.conf> --vv
2016/08/11 14:53:36 [verbose] no config file found at /etc/kong.conf
2016/08/11 14:53:36 [verbose] no config file found at /etc/kong/kong.conf
2016/08/11 14:53:36 [debug] admin_listen = "0.0.0.0:8001"
2016/08/11 14:53:36 [debug] database = "postgres"
2016/08/11 14:53:36 [debug] log_level = "notice"
[...]
```

## 环境变量

当从配置文件中加载属性时，Kong也会查找相同名称的环境变量。这允许您通过环境变量完全配置Kong。

所有环境变量的前缀 `KONG_` ，大写并带有与设置相同的名称将覆盖默认配置。
 例如：

```properties
log_level = debug # in kong.conf
```

会被如下设置覆盖：

```bash
$ export KONG_LOG_LEVEL=error
```

## 定制Nginx配置和嵌入Kong

调整Nginx配置是设置您的Kong实例的一个重要部分，因为它允许您优化其性能，或者将Kong嵌入到已经运行的OpenResty实例中。

- 自定义Nginx配置

Kong可以用 `--nginx-conf` 的参数启动，重新加载和重新启动，该参数必须指定Nginx配置模板。这样的模板使用了Penlight模板引擎，它是使用给定的Kong配置编译的，然后在开始Nginx之前被保存到您的Kong前缀目录中。

默认模板可以在<https://github.com/Kong/kong/tree/master/kong/templates>
 找到。它分为两个Nginx配置文件：`nginx.lua` 和 `nginx_kong.lua`。前者是运行Kong的最低的配置要求，其会包括后者。当Kong开始运行时，在开始Nginx之前，它将这两个文件复制到前缀目录中，看起来是这样的：

```bash
/usr/local/kong
├── nginx-kong.conf
├── nginx.conf
```

如果您希望在Nginx配置中包含其他服务模块，或者您必须调整不受Kong配置影响的全局设置，这里有一个建议：

```nginx
# ---------------------
# custom_nginx.template
# ---------------------

worker_processes ${{NGINX_WORKER_PROCESSES}}; # can be set by kong.conf
daemon ${{NGINX_DAEMON}};                     # can be set by kong.conf

pid pids/nginx.pid;                      # this setting is 强制的
error_log logs/error.log ${{LOG_LEVEL}}; # can be set by kong.conf

events {
    use epoll; # custom setting
    multi_accept on;
}

http {
    # include default Kong Nginx config
    include 'nginx-kong.conf';

    # custom server
    server {
        listen 8888;
        server_name custom_server;

        location / {
          ... # etc
        }
    }
}
```

你可以这样启动Kong：

```shell
$ kong start -c kong.conf --nginx-conf custom_nginx.template
```

如果您希望自定义Kong的Nginx子配置文件，最终添加其他Lua处理程序或定制`lua_*`指令，您可以在`custom_nginx.template`内联`nginx_kong.lua`这个配置。模板示例文件如下:

```nginx
# ---------------------
# custom_nginx.template
# ---------------------

worker_processes ${{NGINX_WORKER_PROCESSES}}; # can be set by kong.conf
daemon ${{NGINX_DAEMON}};                     # can be set by kong.conf

pid pids/nginx.pid;                      # this setting is mandatory
error_log logs/error.log ${{LOG_LEVEL}}; # can be set by kong.conf

events {}

http {
  resolver ${{DNS_RESOLVER}} ipv6=off;
  charset UTF-8;
  error_log logs/error.log ${{LOG_LEVEL}};
  access_log logs/access.log;

  ... # etc
}
```

- 在OpenResty里嵌入Kong

如果您正在运行您自己的OpenResty服务器，您也可以通过使用`include`指令（类似于上一节的示例）来轻松地嵌入Kong。如果您有一个有效的顶级NGINX配置，那么就可以简单的引入Kong的配置：

```nginx
# my_nginx.conf

http {
    include 'nginx-kong.conf';
}
```

你可以像这样开始你的实例：

```bash
$ nginx -p /usr/local/openresty -c my_nginx.conf
```

这样Kong就会运行在你的实例中。（Kong的配置在`nginx-kong.conf`里）

- Kong为网站和你的api提供服务

API提供者的一个常见用例是让Kong在代理端口上同时服务于一个网站和API本身——在生产中有`80`或`443`。例如：`https://my-api.com`（网站）和`https://my-api.com/api/v1`（API）。

为了实现这一点，我们不能简单地声明一个新的虚拟服务模块，就像我们在上一节中所做的那样。一个好的解决方案是使用一个定制的Nginx配置模板，该模板可以内联`nginx_kong.lua`。添加一个新的`location`块，与Kong代理`location`块一起服务于网站：

```nginx
# ---------------------
# custom_nginx.template
# ---------------------

worker_processes ${{NGINX_WORKER_PROCESSES}}; # can be set by kong.conf
daemon ${{NGINX_DAEMON}};                     # can be set by kong.conf

pid pids/nginx.pid;                      # this setting is mandatory
error_log logs/error.log ${{LOG_LEVEL}}; # can be set by kong.conf
events {}

http {
  # here, we inline the contents of nginx_kong.lua
  charset UTF-8;

  # any contents until Kong's Proxy server block
  ...

  # Kong's Proxy server block
  server {
    server_name kong;

    # any contents until the location / block
    ...

    # here, we declare our custom location serving our website
    # (or API portal) which we can optimize for serving static assets
    location / {
      root /var/www/my-api.com;
      index index.htm index.html;
      ...
    }

    # Kong's Proxy location / has been changed to /api/v1
    location /api/v1 {
      set $upstream_host nil;
      set $upstream_scheme nil;
      set $upstream_uri nil;

      # Any remaining configuration for the Proxy location
      ...
    }
  }

  # Kong's Admin server block goes below
}
```

## 配置属性详解

可以新建配置文件`/etc/kong/kong.conf`进行添加修改。

### 常规属性

**prefix**

工作目录。相当于Nginx的前缀路径，包含临时文件和日志。每个流程必须有一个单独的工作目录。

默认：`/usr/local/kong`

**log_level**

Nginx服务器的日志级别。可以在`<prefix>/logs/error.log`
 请参阅 <http://nginx.org/en/docs/ngx_core_module.html#error_log>，以获得公认的值列表。

默认：`notice`

**proxy_access_log**

代理端口请求访问日志的路径。设置为`off`以禁用日志代理请求。如果这个值是相对路径，那么它将被放置于前缀路径之下。

默认：`logs/access.log`

**proxy_error_log**

代理端口请求错误日志的路径。这些日志的粒度由`log_level`指令进行调整。

默认：`logs/error.log`

**admin_access_log**

Admin API的路径请求访问日志。设置为`off`以禁用Admin API请求日志。如果这个值是相对路径，那么它将被放置于前缀路径之下。

默认：`logs/admin_access.log`

**admin_error_log**

Admin API请求错误日志的路径。这些日志的粒度由log_level指令进行调整。

默认：`logs/error.log`

**custom_plugins**

这个节点应该加载的附加插件的逗号分隔列表。使用这个属性来加载与Kong不捆绑的定制插件。插件将从`kong.plugins.{name}.*`命名空间加载。

默认：`none`
 示例：`my-plugin,hello-world,custom-rate-limiting`

**anonymous_reports**

发送匿名的使用数据，比如错误堆栈跟踪，以帮助改进Kong。

默认：`on`



------



### Nginx属性

**proxy_listen**

代理服务侦听的地址和端口的逗号分隔的列表。代理服务是Kong的公共入口点，它代理从您的使用者到您的后端服务的流量。这个值接受IPv4、IPv6和主机名。

可以为每一对指定一些后缀：

-  `ssl` 将要求通过启用TLS的特定地址/端口进行所有连接。
-  `http2` 允许客户端打开http/2连接到Kong的代理服务
- 最后, `proxy_protocol` 将为给定的地址/端口启用代理协议。

这个节点的代理端口，启用了“控制面板”模式（没有流量代理功能），可以配置连接到同一数据库的节点集群。

查看 <http://nginx.org/en/docs/http/ngx_http_core_module.html#listen> 用于描述这个和其他`*_listen`值的接受格式。

默认：`0.0.0.0:8000, 0.0.0.0:8443 ssl`
 示例：`0.0.0.0:80, 0.0.0.0:81 http2, 0.0.0.0:443 ssl, 0.0.0.0:444 http2 ssl`

**admin_listen**

管理接口监听的地址和端口的逗号分隔的列表。Admin接口是允许您配置和管理Kong的API。对该接口的访问应该仅限于Kong管理员。这个值接受IPv4、IPv6和主机名。可以为每一对指定一些后缀：

-  `ssl` 将要求通过启用TLS的特定地址/端口进行所有连接。
-  `http2` 允许客户端打开http/2连接到Kong的代理服务
- 最后, `proxy_protocol` 将为给定的地址/端口启用代理协议。

这个值可以被设置为`off`，从而禁用这个节点的Admin接口，从而使“数据面板”模式（没有配置功能）从数据库中拉出它的配置更改。

默认：`127.0.0.1:8001, 127.0.0.1:8444 ssl`
 示例：`127.0.0.1:8444 http2 ssl`

**nginx_user**

定义工作进程使用的用户和组凭据。如果省略组，则使用名称与用户名相同的组。

默认：`nobody nobody`
 示例：`nginx www`

**nginx_worker_processes**

确定Nginx生成的工作进程的数量。请参阅<http://nginx.org/en/docs/ngx_core_module.html#worker> 流程，以便详细使用该指令和对已接受值的描述。

默认值:`auto`

**nginx_daemon**

确定Nginx是否会作为守护进程或前台进程运行。主要用于开发或在Docker环境中运行Kong。

查阅 <http://nginx.org/en/docs/ngx_core_module.html#daemon>.

默认：`on`

**mem_cache_size**

数据库实体内存缓存的大小。被接受的单位是k和m，最低推荐值为几个MBs。

默认：`128m`

**ssl_cipher_suite**

定义Nginx提供的TLS密码。可接受的值`modern`, `intermediate`, `old`, or `custom`。请参阅 <https://wiki.mozilla.org/Security/Server_Side_TLS>
 ，了解每个密码套件的详细描述。

默认值：`modern`

**ssl_ciphers**

定义一个由Nginx提供的LTS ciphers的自定义列表。这个列表必须符合`openssl ciphers`定义的模式。如果`ssl_cipher_suite`不是`custom`，那么这个值就会被忽略。

默认值：`none`

**ssl_cert**

启用SSL时，`proxy_listen`的SSL证书的绝对路径。

默认值：`none`

**ssl_cert_key**

启用SSL时，`proxy_listen`的SSL key的绝对路径。

默认值：`none`

**client_ssl**

当代理请求时，确定Nginx是否应该发送客户端SSL证书。

默认值：`off`

**client_ssl_cert**

如果启用了client_ssl，用于`proxy_ssl_certificate`配置的客户端SSL证书的绝对路径。注意，这个值是静态地在节点上定义的，并且当前不能在每个api的基础上配置。

默认值：`none`

**client_ssl_cert_key**

如果启用了client_ssl，用于`proxy_ssl_certificate_key`配置的客户端SSL证书的绝对路径。注意，这个值是静态地在节点上定义的，并且当前不能在每个api的基础上配置。

默认值：`none`

**admin_ssl_cert**

启用了SSL后， `admin_listen` 的SSL证书的绝对路径。

默认值：`none`

**admin_ssl_cert_key**

启用了SSL后， `admin_listen` 的SSL key的绝对路径。

默认值：`none`

**upstream_keepalive**

在每个工作进程，设置缓存中保存的upstream服务的空闲keepalive连接的最大数量。当超过这个数字时，会关闭最近最少使用的连接。

默认值：`60`

**server_tokens**

在错误页面，和`Server`或`Via`（如果请求被代理）的响应头字段，启用或禁用展示Kong的版本。

默认值：`on`

**latency_tokens**

在`X-Kong-Proxy-Latency`和`X-Kong-Upstream-Latency`响应头字段中，启用或禁用展示Kong的潜在信息。

默认值：`on`

**trusted_ips**

定义可信的IP地址块，使其知道如何发送正确的 `X-Forwarded-*` 头部信息。来自受信任的ip的请求使Kong转发他们的 `X-Forwarded-*` headers upstream。不受信任的请求使Kong插入自己的 `X-Forwarded-*` headers。

该属性还在Nginx配置中设置 `set_real_ip_from`  指令（s）。它接受相同类型的值（CIDR块），但它是一个逗号分隔的列表。

如果相信 *all* /!\ IPs，请把这个值设为`0.0.0.0/0,::/0`。

如果特殊值`unix:`被指定了，所有的unix域套接字都将被信任。

查阅  [the Nginx docs](http://nginx.org/en/docs/http/ngx_http_realip_module.html#set_real_ip_from) 了解 更详细的`set_real_ip_from`配置资料。

Default: `none`

**real_ip_header**

定义请求头字段，它的值将被用来替换客户端地址。在Nginx配置中使用相同名称的指令 [ngx_http_realip_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html) 设置该值。

如果这个值接收到 `proxy_protocol`，那么 `proxy_protocol` 参数将被附加到Nginx模板的 `listen` 指令中。

查阅 [the Nginx docs](http://nginx.org/en/docs/http/ngx_http_realip_module.html#real_ip_header) 寻找更详细的描述。

默认值： `X-Real-IP`

**real_ip_recursive**

该值设置了Nginx配置中同名的  [ngx_http_realip_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html) 指令。

查阅 [the Nginx docs](http://nginx.org/en/docs/http/ngx_http_realip_module.html#real_ip_recursive) 寻找更详细的描述。

默认值： `off`

**client_max_body_size**

指定在 Content-Length 的请求头中，定义Kong代理的请求的最大被允许的请求体大小。如果请求超过这个限度，Kong将返回413(请求实体太大)。将该值设置为0将禁用检查请求体的大小。

提示: 查阅关于  [the Nginx docs](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size) 这个参数的进一步描述。数值可以用`k`或`m`后缀，表示限制是千字节，还是兆字节。

默认值：`0`

**client_body_buffer_size**

定义读取请求主体的缓冲区大小。如果客户端请求体大于这个值，则阀体将被缓冲到磁盘。请注意，当阀体被缓冲到磁盘的时候，访问或操纵请求主体可能无法工作，因此最好将这个值设置为尽可能高的值。（例如，将其设置为`client_max_body_size`，以迫使请求体保持在内存中）。请注意，高并发性环境需要大量的内存分配来处理许多并发的大型请求体。

提示: 查阅关于 [the Nginx docs](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size) 这个参数的进一步描述。数值可以用`k`或`m`后缀，表示限制是千字节，还是兆字节。

默认值：`8k`

**error_default_type**

当请求Accept标头丢失时，使用默认的MIME类型，且Nginx为这个请求返回一个错误。可接受的值包括 `text/plain`, `text/html`, `application/json`, 和`application/xml`.

默认值：`text/plain`



------



### 数据存储属性

Kong将存储所有的数据（如api、消费者和插件）到Cassandra或PostgreSQL。

属于同一集群的所有Kong节点都必须连接到同一个数据库。

> 从Kong0.12.0开始：
>  PostgreSQL 9.4支持应该被认为是弃用。鼓励用户升级到9.5+
>  应该考虑支持Cassandra 2.1的支持。鼓励用户升级到2.2+

**database**

确定这个节点将使用哪个PostgreSQL或Cassandra作为它的数据存储。可以设置为：`postgres` 和 `cassandra` 和 `off`。

默认值：`postgres`

**Postgres settings**

| 名称          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| pg_host       | Postgres服务器的主机                                         |
| pg_port       | Postgres服务器的端口                                         |
| pg_user       | Postgres用户                                                 |
| pg_password   | Postgres用户的密码                                           |
| pg_database   | 数据库连接。必须存在                                         |
| pg_ssl        | 启用SSL连接到服务器                                          |
| pg_ssl_verify | 如果启用了`pg_ssl`，则切换服务器证书验证。看到`lua_ssl_trusted_certificate`设置。 |

**Cassandra settings**

| 名称                               | 描述                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| cassandra_contact_points           | 指向您的Cassandra集群的链接点列表，使用逗号分割。            |
| cassandra_port                     | 你的节点监听的端口                                           |
| cassandra_keyspace                 | 在集群中使用的关键空间。如果不存在，就会被创建。             |
| cassandra_consistency              | 在阅读/写作时使用一致性设置。                                |
| cassandra_timeout                  | 读取/写入 超时（ms）时间。                                   |
| cassandra_ssl                      | 启用SSL连接到节点。                                          |
| cassandra_ssl_verify               | 如果启用了`cassandra_ssl`，则切换服务器证书验证。查看 `lua_ssl_trusted_certificate` 设置。 |
| cassandra_username                 | 使用PasswordAuthenticator时的用户名。                        |
| cassandra_password                 | 在使用PasswordAuthenticator时的密码。                        |
| cassandra_consistency              | 在读取/写入Cassandra集群时使用一致性设置。                   |
| cassandra_lb_policy                | 当在您的Cassandra集群中分布查询时使用负载平衡策略。可设置为 `RoundRobin` 和 `DCAwareRoundRobin` 。如果您使用的是多数据中心集群，则后者更好。如果是这样，还要设置 `cassandra_local_datacenter`。 |
| cassandra_local_datacenter         | 在使用`DCAwareRoundRobin`政策时，必须指定本地（最近）的集群名称到这个Kong节点。 |
| cassandra_repl_strategy            | 如果第一次创建密钥空间，请指定复制策略。                     |
| cassandra_repl_factor              | 为`SimpleStrategy`指定一个复制因子。                         |
| cassandra_data_centers             | 为`NetworkTopologyStrategy`(网络拓扑策略)指定数据中心。      |
| cassandra_schema_consensus_timeout | Cassandra节点之间同步scheme的超时（ ms）时间。这个值只在数据迁移期间使用。 |



------



### 数据缓存属性

为了避免与数据存储不必要的通信，Kong可配置缓存实体（比如api、消费者、凭证等等）的间隔时间。如果缓存实体被更新，它也会处理也会失效。

本节介绍关于配置Kong此类配置实体缓存。

**db_update_frequency**

频率（以秒为单位），用于检查带有数据存储的更新实体。当节点通过Admin API创建、更新或删除实体时，其他节点需要等待下一次轮询（由这个值配置），以清除旧的缓存实体并开始使用新的缓存。

默认值：5 seconds

**db_update_propagation**

在数据存储中为实体所花费的时间（以秒为单位）被传播到另一个数据中心的副本节点。当在分布式环境中，比如多数据中心Cassandra集群时，这个值应该是Cassandra将一行传播到其他数据中心的最大秒数。当设置了该值，该属性将增加Kong传播实体变更所花费的时间。单数据中心设置或PostgreSQL服务器不应该受到这样的延迟，并且这个值可以安全地设置为0。

默认值: 0 seconds

**db_cache_ttl**

该节点数据存储实体缓存的生存时间（以秒为单位）。数据库遗漏（没有实体）也会根据这个设置进行缓存。如果设置为0，那么这种缓存的实体/遗漏永远不会过期。

默认值：3600 seconds（1小时）



------



### DNS解析属性

Kong将把主机名解析为 `SRV` 或 `A` 记录（按照该顺序，`CNAME` 记录将在此过程中被取消）。如果一个名称被解析为`SRV`记录，它会通过从DNS服务器接收到端口以覆盖给定的端口号。

DNS选项`SEARCH`和`NDOTS`（来自/etc/resolv.conf 文件）将被用于将短名称扩展到完全限定的名称。因此，它将首先尝试完整 `SEARCH` `SRV`类型的列表，如果失败，它将会尝试`SEARCH` `A`记录列表，等等。

在`ttl`的持续时间内，内部DNS解析器将对DNS记录的条目上做负载均衡请求。对于`SRV`记录，可以设置权重，但是它只会使用记录中最低优先级字段条目。

**dns_resolver**

设置域名服务器列表，使用逗号分隔。格式如： `ip[:port]` 。如果没有制定域名服务器，name就使用本地 `resolv.conf` 文件。端口默认为53。可以使用IPv4和IPv6地址。

默认值： none

**dns_hostsfile**

要使用的主机文件。这个文件只被读取一次，然后会存储在内存中。要在修改后想再次读取该文件，必须重新加载Kong。

默认值：`/etc/hosts`

**dns_order**

解决不同记录类型的顺序。`LAST`类型指的是最后一次成功的查找的类型（对于指定的名称）。格式是一个（大小写不敏感）逗号分隔的列表。

默认值： `LAST`,`SRV`,`A`,`CNAME`

**dns_stale_ttl**

定义在缓存中保存DNS记录的`TTL`时间。当新的DNS记录在后台获取时，这个值将被使用。陈旧的数据将从记录的过期时间使用，直到刷新查询完成，或者`dns_stale_ttl`的秒数已经过去。

默认值：`4`

**dns_not_found_ttl**

空DNS响应和 "(3) name error" 响应的TTL时间（以秒为单位）

默认值：`30`

**dns_error_ttl**

错误响应的TTL时间（以秒为单位）

默认值：`1`

**dns_no_sync**

如果启用了，那么在cache-miss时，每个请求都会触发自己的dns查询。当为相同的名称/类型禁用多个请求时，将同步到单个查询。

默认值： off



------



### 开发与其他属性

从lua-nginx-module继承的附加设置，可以更灵活和更高级的使用。
 有关更多信息，请参见lua-nginx-module文档：<https://github.com/openresty/lua-nginx-module>



------



**lua_ssl_trusted_certificate**

在PEM格式的Lua cosockets的证书权威文件的绝对路径。该证书将用于验证Kong的数据库连接，当启用`pg_ssl_verify`或`cassandra_ssl_verify`时。

详情查阅：<https://github.com/openresty/lua-nginx-module#lua_ssl_trusted_certificate>

默认值： none



------



**lua_ssl_verify_depth**

在Lua cosockets使用的服务器证书链中设置验证深度，通过`lua_ssl_trusted_certificate` 设置。

这包括为Kong的数据库连接配置的证书。

详情查阅： <https://github.com/openresty/lua-nginx-module#lua_ssl_verify_depth>

默认值: `1`



------



**lua_package_path**

设置Lua模块搜索路径（LUA_PATH）。在默认搜索路径中，开发或使用不存储的自定义插件时非常有用。

详情查阅：<https://github.com/openresty/lua-nginx-module#lua_package_path>

默认值： none



------



**lua_package_cpath**

设置Lua C模块搜索路径（LUA_CPATH）。

详情查阅：<https://github.com/openresty/lua-nginx-module#lua_package_cpath>

默认值： none



------



**lua_socket_pool_size**

指定与每个远程服务器相关联的每个cosocket连接池的大小限制。

详情查阅：<https://github.com/openresty/lua-nginx-module#lua_socket_pool_size>

默认值：`30`

---

作者：DreamsonMa

链接：https://www.jianshu.com/p/f3b1699777d6

来源：简书

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。