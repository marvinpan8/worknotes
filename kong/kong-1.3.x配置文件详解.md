# [Kong 1.3.x 配置文件详解](https://linuxops.org/blog/kong/config.html)

2018-09-24 /  Linuxops

### 重要属性-marvinpan

```properties
#决定了Kong集群节点在性能和一致性间的平衡
db_update_frequency = 5
#该节点数据存储实体缓存的生存时间(秒)
db_cache_ttl = 600 
anonymous_reports = off 
headers = off
dns_resolver=
```

## 一、前言

Kong配置文件是Kong服务的核心文件，它配置了Kong以怎么的方式运行，并且依赖于这个配置生成Nginx的配置文件，本文通过解读Kong配置文件，以了解Kong的运行和配置。

在成功安装Kong以后，会有一个名为`kong.conf.default`默认的配置文件示例,如果是通过包管理器安装的，通常位于`/etc/kong/kong.conf.default`，我们要将其复制为`kong.conf`以便于我们修改使用他。

在Kong的配置文件中，约定了以下的几条规则：

- 配置文件中以`#`开头的行均为注释行，程序不会读取这些内容。
- 在官方提供的默认配置文件中，以`#`开头的有值的配置项目均为默认配置。
- 所有的配置项，均可以在系统环境变量中配置，但是必须要加上`KONG_`为前缀。
- 值为布尔型的配置，可以使用`on`/`off`或者`true`/`false`。
- 值为列表的，必须使用半角逗号分割。

Kong的配置，大概分为几种，分别是：

- 常规配置：配置服务运行目录，插件加载，日志等等
- NGINX配置：配置Nginx注入，例如监听IP和端口配置等等，用于Kong在启动的时候生成Nginx配置文件
- 数据库存储配置：配数据库类型，地址、用户名密码等等信息
- 数据库缓存配置：配置数据的缓存规则，Kong会缓存诸如API信息、用户、凭证等信息，以减少访问数据库次数提高性能
- DNS解析器配置：默认情况会使用系统设置，如`hosts`和`resolv.conf`的配置，你也可以通过DNS的解析器配置来修改
- 其他杂项配置：继承自lua-nginx模块的其他设置允许更多的灵活性和高级用法。

下面我们一个模块一个模块解释一下各项的配置。

## 二、常规配置

在常规配置中，主要是控制Kong一些运行时的一些配置，主要有如下配置：

| 配置项            | 默认值                | 说明                                                         |
| ----------------- | --------------------- | ------------------------------------------------------------ |
| prefix            | /usr/local/kong/      | 配置Kong的工作目录，相当于Nginx的工作目录，这个目录存放运行时的临时文件和日志，包括Kong启动的时候自动生成的Nginx的配置文件。 每一个Kong经常必须有一个单独的工作目录 |
| log_level         | notice                | Nginx的日志级别。日志存放/logs/error.log                     |
| proxy_access_log  | logs/access.log       | 代理端口请求的日志文件，可以设置为`off`来关闭日志的记录，也可以通过设置绝对路径也可以设置相对路径。 如果设置了相对路径，则日志文件会保存在的目录下 |
| proxy_error_log   | logs/error.log        | 代理端口请求的错误日志文件，可以设置为`off`来关闭日志的记录，也可以通过设置绝对路径也可以设置相对路径。 如果设置了相对路径，则日志文件会保存在的目录下 |
| admin_access_log  | logs/admin_access.log | Kong管理的API端口请求的日志文件，可以设置为`off`来关闭日志的记录，也可以通过设置绝对路径也可以设置相对路径。 如果设置了相对路径，则日志文件会保存在的目录下 |
| admin_error_log   | logs/error.log        | Kong管理的API端口请求的错误日志文件，可以设置为`off`来关闭日志的记录，也可以通过设置绝对路径也可以设置相对路径。 如果设置了相对路径，则日志文件会保存在的目录下 |
| plugins           | bundled               | Kong启动的时候加载的插件，如果多个必须要使用半角逗号分割。默认情况下，只有捆绑官方发行版本的插件通过 `bundled` 这个值来加载。 加载插件只是Kong在启动的时候载入插件的代码，但是并不会使用它，如果要使用他，还必须要通过管理API来配置 当然，如果你不想加载任何插件，可以使用`off`来关闭它，值得强调的一点`bundled`值可以和其他插件名称一起使用，`bundled`并不是一个插件名称，它代表了随官方发行的所有插件。 |
| anonymous_reports | on                    | 如果Kong进程发生了错误，会以匿名的方式将错误提交给Kong官方， 以帮助改善Kong。 |

在常规的配置中，主要配置了Kong运行的目录日志等信息。

无论如何，配置的文件或者目录Kong必须要用权限访问，否则会报错。

## 三、Nginx注入配置

Kong基于Nginx，当然需要配置Nginx来满足Kong的运行要求，Kong提供了Nginx的注入配置，使得我们更轻松控制。

在Nginx注入配置中，有如下配置：

| 配置项                  | 默认值                             | 说明                                                         |
| ----------------------- | ---------------------------------- | ------------------------------------------------------------ |
| proxy_listen            | 0.0.0.0:8000, 0.0.0.0:8443 ssl     | 配置Kong代理监听的地址和端口，这个是Kong的入口，API调用都将通过这个端口请求。这个配置和Ngxin中的配置一致，通过`SSL`可以指定接受https的请求 支持IPv4和IPv6 |
| admin_listen            | 127.0.0.1:8001, 127.0.0.1:8444 ssl | 配置Kong的管理API监听的端口，和proxy_listen配置一样，但是这个配置不建议监听在公网IP上。 |
| nginx_user              | nobody nobody                      | 配置Nginx的用户名和用户组，和Nginx的配置规则一样             |
| nginx_worker_processes  | auto                               | 设置Nginx的进程书，通常等于CPU核心数                         |
| nginx_daemon            | on                                 | 是否以daemon的方式运行Ngxin                                  |
| mem_cache_size          | 128m                               | 内存的缓存大小，可以使用`k`和`m`为单位                       |
| ssl_cipher_suite        | modern                             | 定义Nginx提供的TLS密码，可以配置的值有：`modern`,`intermediate`, `old`, `custom`. |
| ssl_ciphers             |                                    | 定义Nginx提供的TLS密码的列表，参考Nginx的配置                |
| ssl_cert                |                                    | 配置SSL证书的crt路径，必须是要绝对路径                       |
| ssl_cert_key            |                                    | 设置SSL证书的key文件，必须是绝对路径                         |
| client_ssl              | off                                | .....                                                        |
| client_ssl_cert         |                                    | .....                                                        |
| client_ssl_cert_key     |                                    | .....                                                        |
| admin_ssl_cert          |                                    | .....                                                        |
| admin_ssl_cert_key      |                                    | .....                                                        |
| headers                 | server_tokens, latency_tokens      | 设置在相应客户端时候应该注入的头部，可以设置的值如下：  - `server_tokens`: 注入'Via'和'Server'头部. - `latency_tokens`: 注入'X-Kong-Proxy-Latency'和'X-Kong-Upstream-Latency' 头部. - `X-Kong-<header-name>`: 只有在适当的时候注入特定的头部 这个配置可以被设置为`off`。当然，即便设置了`off`以后，插件依然可以注入头部 |
| trusted_ips             |                                    | 定义可信的IP地址段，通常不建议在此处限制请求，应该再插件中过滤 |
| real_ip_header          | X-Real-IP                          | 获取客户端真实的IP，将值通过同步的形式传递给后端             |
| real_ip_recursive       | off                                | 这个值在Nginx配置中设置了同名的ngx_http_realip_module指令    |
| client_max_body_size    | 0                                  | 配置Nginx接受客户端最大的body长度，如果超过此配置 将返回413。 设置为`0`则不检查长度 |
| client_body_buffer_size | 8k                                 | 设置读取缓冲区大小，如果超过内存缓冲区大小，那么NGINX会缓存在磁盘中，降低性能。 |
| error_default_type      | text/plain                         | 当请求' Accept '头丢失，Nginx返回请求错误时使用的默认MIME类型。可以配置的值为： `text/plain`,`text/html`, `application/json`, `application/xml`. |

在Nginx注入配置中，配置了Nginx的基本的参数，这些参数大部分和NGINX的配置值是一样的，可以通过Nginx的配置文档了解一下。

## 四、 数据库存储配置

数据库配置的模块配置数据库相关的连接信息等等。主要有如下配置：

| 配置项                             | 默认值         | 说明                                                         |
| ---------------------------------- | -------------- | ------------------------------------------------------------ |
| database                           | postgres       | 设置数据库类型，Kong支持两种数据库，一种是postgres，一种是cassandra |
| PostgreSQL配置                     |                | 如果`database`设置为`postgres`以下配置生效                   |
| pg_host                            | 127.0.0.1      | 设置PostgreSQL的连接地址                                     |
| pg_port                            | 5432           | 设置PostgreSQL的端口                                         |
| pg_user                            | kong           | 设置PostgreSQL的用户名                                       |
| pg_password                        |                | 设置PostgreSQL的密码                                         |
| pg_database                        | kong           | 设置数据库名称                                               |
| pg_ssl                             | off            | 是否开启ssl连接                                              |
| pg_ssl_verify                      | off            | 如果启用了' pg_ssl '，则切换服务器证书验证。                 |
| cassandra配置                      |                | 如果`database`设置为`cassandra`以下配置生效                  |
| cassandra_contact_points           | 127.0.0.1      | .....                                                        |
| cassandra_port                     | 9042           | .....                                                        |
| cassandra_keyspace                 | kong           | .....                                                        |
| cassandra_timeout                  | 5000           | .....                                                        |
| cassandra_ssl                      | off            | .....                                                        |
| cassandra_ssl_verify               | off            | .....                                                        |
| cassandra_username                 | kong           | .....                                                        |
| cassandra_password                 |                | .....                                                        |
| cassandra_consistency              | ONE            | .....                                                        |
| cassandra_lb_policy                | RoundRobin     | .....                                                        |
| cassandra_local_datacenter         |                | .....                                                        |
| cassandra_repl_strategy            | SimpleStrategy | .....                                                        |
| cassandra_repl_factor              | 1              | .....                                                        |
| cassandra_data_centers             | dc1:2,dc2:3    | .....                                                        |
| cassandra_schema_consensus_timeout | 10000          | .....                                                        |

> 推荐使用PostgreSQL数据库作为生产环境的存储，PostgreSQL具有良好的性能和稳定性，是一个非常优秀的开源数据库。

## 五、 数据库缓存配置

在上一节中，配置了Kong持久化存储，显然如果每次的请求都需要去查询数据库中的相关信息那无疑是非常消耗资源，性能和稳定性也会大大降低，作为一个API网关肯定是不能忍的，解决这个问题的办法就是缓存，Kong将数据缓存在内存中，这样会大大提高性能，本节介绍Kong的缓存配置。

| 配置项                | 默认值 | 说明                                                         |
| --------------------- | ------ | ------------------------------------------------------------ |
| db_update_frequency   | 5      | 节点更新数据库的时间，以秒为单位。 这个配置设置了节点查询数据库的时间，假如有3台Kong服务器节点ABC，如果再A节点增加了一个API网关，那么B和C节点最多需要等待`db_update_frequency`时间才能被更新到。 |
| db_update_propagation | 0      | 数据库节点的更新时间。 如果使用了Cassandra数据库集群，那么如果数据库有更新，最多需要`db_update_propagation`时间来同步所有的数据库副本。 如果使用PostgreSQL或者单数据库，这个值可以被设置为0 |
| db_cache_ttl          | 0      | 缓存生效时间，单位秒。如果设置为0表示永不过期 Kong从数据库中读取数据并且缓存，在ttl过期后会删除这个缓存然后再一次读取数据库并缓存 |
| db_resurrect_ttl      | 30     | 缓存刷新时间，单位秒。当数据存储中的陈旧实体无法刷新时(例如，数据存储不可访问)，应该对其进行恢复。当这个TTL过期时，将尝试刷新陈旧的实体。 |

## 六、 DNS解析器配置

默认情况下，DNS解析器将使用标准配置文件`/etc/hosts`和`/etc/resolv.conf`。如果设置了环境变量`LOCALDOMAIN`和`RES_OPTIONS`，那么后一个文件中的设置将被覆盖。

| 配置项            | 默认值           | 说明                                                         |
| ----------------- | ---------------- | ------------------------------------------------------------ |
| dns_resolver      |                  | 配置DNS服务器列表，用半角逗号分割，每个条目使用ip[:port]的格式，这个配置仅提供给Kong使用，不会覆盖节点系统的配置，如果没有配置则使用系统的设置。接受IPv4和IPv6的地址。 |
| dns_hostsfile     | /etc/hosts       | 配置Kong的hosts文件，这个配置同样仅提供给Kong使用，不会覆盖节点系统的配置。 需要说明的是这个文件仅读取一次，读取的内容会缓存再内存中，如果修改了此文件，必须要重启Kong才能生效。 |
| dns_order         | LAST,SRV,A,CNAME | 解析不同记录类型的顺序。“LAST”类型表示最后一次成功查找的类型(用于指定的名称) |
| dns_stale_ttl     | 4                | 配置DNS记录缓存过期时间                                      |
| dns_not_found_ttl | 30               | 这个配置值不知道该如何理解？？                               |
| dns_error_ttl     | 1                | .....                                                        |
| dns_no_sync       | off              | 如果启用了该项，那么在DNS缓存过期之后，每一次请求都会发起DNS查询。在禁用此项时，那么相同的域名多次请求会同步到一个查询中共享返回值。 |

在DNS配置中，我们基本上不需要更改，官网的配置给出了最优的配置。如果我们需要在`host`文件中定义后端绑定的域名，一定要在编辑`hosts`文件后重载Kong的配置，或者重启Kong，无论`hosts`的文件是否是`/etc/hosts`，否则都不会生效的。

## 七、 其他杂项配置

杂项配置基本上关于LUA的配置，如果不熟悉请不要修改，按照官方默认即可。

| 配置项                      | 默认值                | 说明 |
| --------------------------- | --------------------- | ---- |
| lua_ssl_trusted_certificate |                       | .... |
| lua_ssl_verify_depth        | 1                     | .... |
| lua_package_path            | ./?.lua;./?/init.lua; | .... |
| lua_package_cpath           |                       | .... |
| lua_socket_pool_size        | 30                    | .... |