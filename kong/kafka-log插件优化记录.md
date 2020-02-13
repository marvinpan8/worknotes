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