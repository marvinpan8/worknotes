# Kafka-log 插件调试优化记录

#### 直接进入容器调试, 修改后 kong reload 

```bash
cd /usr/local/share/lua/5.1/kong/plugins
### 修改文件增加变量
local ngx = ngx
local kong = kong
local fmt = string.format
```

#### lua table 转string

方案一

```lua
local cjson = require "cjson"
local cjson_encode = cjson.encode
--tab的属性也可以是数组
local tab = {}
tab["Himi"] = "himigame.com"
--数据转json
local cjson = require"cjson"
local jsonData = cjson.encode(tab)
 
print(jsonData)
--打印结果:  {"Himi":"himigame.com"}
 
--json转数据
local data = cjson_encode(jsonData)
 
print(data.Himi)
--打印结果:  himigame.com
```



方案二

```lua
function table2string(obj)
    local lua = ""  
    local t = type(obj)  
    if t == "number" then  
        lua = lua .. obj  
    elseif t == "boolean" then  
        lua = lua .. tostring(obj)  
    elseif t == "string" then  
        lua = lua .. fmt("%q", obj)  
    elseif t == "table" then  
        lua = lua .. "{"  
        for k, v in pairs(obj) do  
            lua = lua .. table2string(k) .. ":" .. table2string(v) .. ","  
        end  
        local metatable = getmetatable(obj)  
        if metatable ~= nil and type(metatable.__index) == "table" then  
            for k, v in pairs(metatable.__index) do  
                lua = lua .. table2string(k) .. ":" .. table2string(v) .. ","  
            end  
        end
        lua = lua .. "}"  
    elseif t == "function" then
        lua = lua .. "function"
    elseif t == "nil" then  
        return nil  
    else  
        kong.log.err("can not table2string a " .. t .. " type.")  
    end  
    return lua  
end
```

增加  log

```lua
kong.log.debug("message====>>>", table2string(message))
```

---

## 缓存

作为网关代理层, 缓存一定是必不可少的一环, 在kong的PDK中, 封装了lua-resty-mlcache.
 kong的缓存分为两级:

- L1: Lua memory cache - 在nginx worker中共享, 可以存储任何Lua值
- L2: Shared memory cache (SHM) - 在nginx node的所有worker中共享. 可以存储任何标量值, 但它需要序列化和反序列化, 所以性能会有所下降

> 注: 从数据库中提取数据后，它将同时存储于上述两级缓存中。现在，如果同一个工作进程再次请求数据，它将从Lua内存缓存检索数据。 如果同一个Nginx节点中的不同工作者请求该数据，它将在SHM中找到数据，对其进行反序列化（并将其存储在自己的Lua内存缓存中），然后将其返回。

一个典型的使用方式如下:

```lua
-- 通过一个唯一值获取Cache Key
cache_key = singletons.dao.keyauth_credentials:cache_key("api_key")
value, err = singletons.cache:get(cache_key ,nil, load_from_db_func, ....)
```

函数: `value, err = cache:get(key, opts?, cb, ...)` , 如果cache没有值(miss), 则会调用函数`cb`, `cb` 必须返回一个返回值, 可返回需要缓存的值或者`nil`, 需要注意的是返回值为`nil`时, get函数依旧会执行缓存, 但我们可以通过`cache:get()`的第二个参数控制缓存的TTL和negative TTL. 如下代码即代表有数据时, 我们缓存600s, 没有数据时, 缓存`nil`结果40s

```lua
value, err = singletons.cache:get(cache_key ,{
            ttl = 600, --如果有数据, 则缓存的时间, 单位:秒
            neg_ttl = 40 -- 如果没有数据, 单位:秒
        }, load_from_db_func, ....)
```

---

## 注意事项

- '%' 用作特殊字符的转义字符，因此 '%.' 匹配点；'%%' 匹配字符 '%'。转义字符 '%'不仅可以用来转义特殊字符，还可以用于所有的非字母的字符。当对一个字符有疑问的时候，为安全起见请使用转义字符转义它。

- `kong.db.consumer_services:select_by_name`  by后面必须是唯一键或者endpoint_key

## 日志格式

```json
{
    "request": {
        "method": "GET",
        "uri": "/get",
        "url": "http://httpbin.org:8000/get",
        "size": "75",
        "querystring": {},
        "headers": {
            "accept": "*/*",
            "host": "httpbin.org",
            "user-agent": "curl/7.37.1"
        },
        "tls": {
            "version": "TLSv1.2",
            "cipher": "ECDHE-RSA-AES256-GCM-SHA384",
            "supported_client_ciphers": "ECDHE-RSA-AES256-GCM-SHA384",
            "client_verify": "NONE"
        }
    },
    "upstream_uri": "/",
    "response": {
        "status": 200,
        "size": "434",
        "headers": {
            "Content-Length": "197",
            "via": "kong/0.3.0",
            "Connection": "close",
            "access-control-allow-credentials": "true",
            "Content-Type": "application/json",
            "server": "nginx",
            "access-control-allow-origin": "*"
        }
    },
    "tries": [
        {
            "state": "next",
            "code": 502,
            "ip": "127.0.0.1",
            "port": 8000
        },
        {
            "ip": "127.0.0.1",
            "port": 8000
        }
    ],
    "authenticated_entity": {
        "consumer_id": "80f74eef-31b8-45d5-c525-ae532297ea8e",
        "id": "eaa330c0-4cff-47f5-c79e-b2e4f355207e"
    },
    "route": {
        "created_at": 1521555129,
        "hosts": null,
        "id": "75818c5f-202d-4b82-a553-6a46e7c9a19e",
        "methods": null,
        "paths": [
            "/example-path"
        ],
        "preserve_host": false,
        "protocols": [
            "http",
            "https"
        ],
        "regex_priority": 0,
        "service": {
            "id": "0590139e-7481-466c-bcdf-929adcaaf804"
        },
        "strip_path": true,
        "updated_at": 1521555129
    },
    "service": {
        "connect_timeout": 60000,
        "created_at": 1521554518,
        "host": "example.com",
        "id": "0590139e-7481-466c-bcdf-929adcaaf804",
        "name": "myservice",
        "path": "/",
        "port": 80,
        "protocol": "http",
        "read_timeout": 60000,
        "retries": 5,
        "updated_at": 1521554518,
        "write_timeout": 60000
    },
    "workspaces": [
        {
            "id":"b7cac81a-05dc-41f5-b6dc-b87e29b6c3a3",
            "name": "default"
        }
    ],
    "consumer": {
        "username": "demo",
        "created_at": 1491847011000,
        "id": "35b03bfc-7a5b-4a23-a594-aa350c585fa8"
    },
    "latencies": {
        "proxy": 1430,
        "kong": 9,
        "request": 1921
    },
    "client_ip": "127.0.0.1",
    "started_at": 1433209822425
}
```

A few considerations on the above JSON object:

- `request` contains properties about the request sent by the client

- `response` contains properties about the response sent to the client

- `tries` contains the list of (re)tries (successes and failures) made by the load balancer for this request

- `route` contains Kong properties about the specific Route requested

- `service` contains Kong properties about the Service associated with the requested Route

- `authenticated_entity` contains Kong properties about the authenticated credential (if an authentication plugin has been enabled)

- `workspaces` contains Kong properties of the Workspaces associated with the requested Route. **Only in Kong Enterprise version >= 0.34**.

- `consumer` contains the authenticated Consumer (if an authentication plugin has been enabled)

- latencies  contains some data about the latencies involved:

  - `proxy` is the time it took for the final service to process the request(是最终服务处理请求所花费的时间，单位ms)
  - `kong` is the internal Kong latency that it took to run all the plugins(是运行所有插件所需的内部Kong延迟，单位ms)
  - `request` is the time elapsed between the first bytes were read from the client and after the last bytes were sent to the client. Useful for detecting slow clients(是从客户端读取的第一个字节之间以及最后一个字节发送到客户端之间经过的时间。用于检测慢速客户端, 一般是前2项之和，单位ms).

- `client_ip` contains the original client IP address

- `started_at` contains the UTC timestamp of when the request has started to be processed.
