# [nginx通过自定义header属性来转发不同的服务](https://www.cnblogs.com/xiao987334176/p/11263649.html)

# 一、背景

因为需要上线灰度发布，只要nginx接收到头部为：

```
wx_unionid:123456
```

就会跳转到另外一个url，比如：

```
127.0.0.1:8080
```

通过配置nginx 匹配请求头wx_unionid 来转发到灰度环境。
核心：客户端自定义的http header，在nginx的配置文件里能直接读取到。
条件：header必须用减号“-”分隔单词，nginx里面会转换为对应的下划线“_”连接的小写单词。



# 二、修改Nginx配置

安装nginx

```
apt-get install -y nginx
```

## 编辑主页

```
cd /etc/nginx/sites-enabled
vim home.conf
```

内容如下：

```
upstream wx {
    server 127.0.0.1:8080;
}

server {  
    listen 8008;
    server_name localhost;

    root html;
    index index.html;

    charset utf-8;
    underscores_in_headers on;
    location / {
        #测试header转发 
        if ($http_wx_unionid = "123456") {
            proxy_pass http://wx;
        }  
    }  
} 
```



## 参数配置说明

underscores_in_headers on：nginx是支持读取非nginx标准的用户自定义header的，但是需要在http或者server下开启header的下划线支持:
比如我们自定义header为wx_unionid,获取该header时需要这样:$http_wx_unionid(一律采用小写，而且前面多了个http_)

如果需要把自定义header传递到下一个nginx：

1. 如果是在nginx中自定义采用 **proxy_set_header X_CUSTOM_HEADER $http_host;**
2. 如果是在用户请求时自定义的header，例如curl –head -H “X_CUSTOM_HEADER: foo” http://domain.com/api/test，则需要通过 **proxy_pass_header X_CUSTOM_HEADER** 来传递



## 编辑调整页

```
vim wx.conf
```

内容如下：



```
server {
    listen 8080;
    server_name localhost;

    root /var/www/html;
    index wx.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

增加测试页面

```
vim /var/www/html/wx.html
```

内容如下：

```
<h1>微信小程序测试平台</h1>
```

 

# 三、测试结果

加自定义头部



```
root@ubuntu:~# curl -v -H 'wx_unionid:123456' 127.0.0.1:8008
* Rebuilt URL to: 127.0.0.1:8008/
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8008 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1:8008
> User-Agent: curl/7.47.0
> Accept: */*
> wx_unionid:123456
> 
< HTTP/1.1 200 OK
< Server: nginx/1.10.3 (Ubuntu)
< Date: Mon, 29 Jul 2019 07:49:46 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 37
< Connection: keep-alive
< Last-Modified: Mon, 29 Jul 2019 06:23:49 GMT
< ETag: "5d3e90f5-25"
< Accept-Ranges: bytes
< 
<h1>微信小程序测试平台</h1>
* Connection #0 to host 127.0.0.1 left intact
```

不加头部

```bash
root@ubuntu:~# curl -v 127.0.0.1:8008
* Rebuilt URL to: 127.0.0.1:8008/
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8008 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1:8008
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.10.3 (Ubuntu)
< Date: Mon, 29 Jul 2019 07:50:13 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 612
< Last-Modified: Tue, 31 Jan 2017 15:01:11 GMT
< Connection: keep-alive
< ETag: "5890a6b7-264"
< Accept-Ranges: bytes
< 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host 127.0.0.1 left intact
```

 

# 四、匹配多条件

现在又多了一个需求，需要对客户端ip为：192.168.0.45，同时也增加自定义头部wx_unionid:123456，才会转发，否则不做转发。

 

[nginx](http://www.ttlsa.com/nginx/)的**配置中不支持if条件的逻辑与&& 逻辑或|| 运算** ，而且**不支持if的嵌套语法**，否则会报下面的错误：nginx: [emerg] invalid condition。

我们可以用变量的方式来间接实现。

要实现的语句：

```
if ($remote_addr ~* "192.168.0.45" && $http_wx_unionid = "123456"){
    proxy_pass http://wx;
}
```

 

如果按照这样来配置，就会报nginx: [emerg] invalid condition错误。

如下：

```
nginx: [emerg] invalid condition "$http_wx_unionid" in /etc/nginx/sites-enabled/home.conf:16
nginx: configuration file /etc/nginx/nginx.conf test failed
```

 

可以这么来实现，如下所示：



```
upstream wx {
    server 127.0.0.1:8080;
}

server {  
    listen 8008;
    server_name localhost;

    root html;
    index index.html;

    charset utf-8;
    underscores_in_headers on;
    location / {
        #测试header转发
        # 标志位
        set $flag 0;
        if ($remote_addr ~* "192.168.0.45"){
             # 末尾追加1，此时$flag=01
             set $flag "${flag}1";
        }
        if ($http_wx_unionid = "123456"){
            # 末尾追加1，此时$flag=011
            set $flag "${flag}1";
        }
        # 当为011时，表示条件都匹配
        if ($flag = "011") {
            proxy_pass http://wx;
        }
    }  
} 
```


重新加载配置

```
nginx -s reload
```

 

再次使用本机测试，增加头部

```bash
root@ubuntu:~# curl -v -H 'wx_unionid:123456' 127.0.0.1:8008
* Rebuilt URL to: 127.0.0.1:8008/
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8008 (#0)
> GET / HTTP/1.1
> Host: 127.0.0.1:8008
> User-Agent: curl/7.47.0
> Accept: */*
> wx_unionid:123456
> 
< HTTP/1.1 200 OK
< Server: nginx/1.10.3 (Ubuntu)
< Date: Mon, 29 Jul 2019 08:01:30 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 612
< Last-Modified: Tue, 31 Jan 2017 15:01:11 GMT
< Connection: keep-alive
< ETag: "5890a6b7-264"
< Accept-Ranges: bytes
< 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host 127.0.0.1 left intact
```

 

登录到192.168.0.45，增加头部测试

```bash
root@docker-reg:~# curl -v -H 'wx_unionid:123456' 192.168.0.162:8008
* Rebuilt URL to: 192.168.0.162:8008/
*   Trying 192.168.0.162...
* Connected to 192.168.0.162 (192.168.0.162) port 8008 (#0)
> GET / HTTP/1.1
> Host: 192.168.0.162:8008
> User-Agent: curl/7.47.0
> Accept: */*
> wx_unionid:123456
> 
< HTTP/1.1 200 OK
< Server: nginx/1.10.3 (Ubuntu)
< Date: Mon, 29 Jul 2019 08:02:01 GMT
< Content-Type: text/html; charset=utf-8
< Content-Length: 37
< Connection: keep-alive
< Last-Modified: Mon, 29 Jul 2019 06:23:49 GMT
< ETag: "5d3e90f5-25"
< Accept-Ranges: bytes
< 
<h1>微信小程序测试平台</h1>
* Connection #0 to host 192.168.0.162 left intact
```

  

 本文参考链接：

<https://blog.csdn.net/qq_25934401/article/details/83113520>

<https://blog.csdn.net/lai0yuan/article/details/80784058>

<https://blog.csdn.net/woshizhangliang999/article/details/51701327>

 