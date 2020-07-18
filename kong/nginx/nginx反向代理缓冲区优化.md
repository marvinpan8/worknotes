# [Nginx反向代理缓冲区优化](https://www.cnblogs.com/xiewenming/p/8023090.html)

## 内容目录

- proxy_buffering
- proxy_buffer_size
- **proxy_buffers**
- proxy_busy_buffers_size
- **proxy_max_temp_file_size** 和proxy_temp_file_write_size

关于缓冲, 主要是合理设置缓冲区大小, 尽量避免缓冲到硬盘时的情况

如果一台代理服务器上面配置了多个域名，可以在每个域名的location区域设置，在这里配置的参数会覆盖nginx.conf的全局配置参数，从而对不同域名的业务需要进行针对性的设置

## proxy_buffering

proxy_buffering这个参数用来控制是否打开后端响应内容的缓冲区，如果这个设置为off，那么proxy_buffers和proxy_busy_buffers_size这两个指令将会失效。 但是无论proxy_buffering是否开启，对proxy_buffer_size都是生效的。

proxy_buffering开启的情况下，nignx会把后端返回的内容先放到缓冲区当中，然后再返回给客户端(边收边传，不是全部接收完再传给客户端)。 临时文件由proxy_max_temp_file_size和proxy_temp_file_write_size这两个指令决定的。

如果proxy_buffering关闭，（比如大图片用户打开一半然后就不显示了）那么nginx会立即把从后端收到的响应内容传送给客户端，每次取的大小为proxy_buffer_size的大小，这样效率肯定会比较低。

注： proxy_buffering启用时，要提防使用的代理缓冲区太大。这可能会吃掉你的内存，限制代理能够支持的最大并发连接数。

## proxy_buffer_size

后端服务器的相应头会放到proxy_buffer_size当中，这个大小默认等于proxy_buffers当中的设置单个缓冲区的大小。 proxy_buffer_size只是响应头的缓冲区，没有必要也跟着设置太大。 proxy_buffer_size最好单独设置，一般设置个4k就够了。

## proxy_buffers

proxy_buffers的缓冲区大小一般会设置的比较大，以应付大网页。 proxy_buffers当中单个缓冲区的大小是由系统的内存页面大小决定的，Linux系统中一般为4k。 proxy_buffers由缓冲区数量和缓冲区大小组成的。总的大小为number*size。

若某些请求的响应过大,则超过*_buffers的部分将被缓冲到硬盘(缓冲目录由*_temp_path指令指定), 当然这将会使读取响应的速度减慢, 影响用户体验. 可以使用proxy_max_temp_file_size指令关闭磁盘缓冲.

## proxy_busy_buffers_size

proxy_busy_buffers_size不是独立的空间，他是proxy_buffers和proxy_buffer_size的一部分。nginx会在没有完全读完后端响应的时候就开始向客户端传送数据，所以它会划出一部分缓冲区来专门向客户端传送数据(这部分的大小是由proxy_busy_buffers_size来控制的，建议为proxy_buffers中单个缓冲区大小的2倍)，然后它继续从后端取数据，缓冲区满了之后就写到磁盘的临时文件中。

## proxy_max_temp_file_size和proxy_temp_file_write_size

临时文件由proxy_max_temp_file_size和proxy_temp_file_write_size这两个指令决定。 proxy_temp_file_write_size是一次访问能写入的临时文件的大小，默认是proxy_buffer_size和proxy_buffers中设置的缓冲区大小的2倍，Linux下一般是8k。

proxy_max_temp_file_size指定当响应内容大于proxy_buffers指定的缓冲区时, 写入硬盘的临时文件的大小. 如果超过了这个值, Nginx将与Proxy服务器同步的传递内容, 而不再缓冲到硬盘. 设置为0时, 则直接关闭硬盘缓冲.

## 配置示例

- 通用网站的配置

```bash
proxy_buffer_size 4k; #设置代理服务器（nginx）保存用户头信息的缓冲区大小
proxy_buffers 4 32k; #proxy_buffers缓冲区，网页平均在32k以下的设置
proxy_busy_buffers_size 64k; #高负荷下缓冲大小（proxy_buffers*2）
proxy_temp_file_write_size 64k;
#设定缓存文件夹大小，大于这个值，将从upstream服务器传
```

- docker registry的配置 这个每次传输至少都是9M以上的内容，缓冲区配置大；

```bash
proxy_buffering on;
proxy_buffer_size 4k; 
proxy_buffers 8 1M;
proxy_busy_buffers_size 2M;
proxy_max_temp_file_size 0
```

## 总结

### buffer工作原理
首先第一个概念是所有的这些`proxy buffer`参数是作用到每一个请求的。每一个请求会安按照参数的配置获得自己的buffer。proxy buffer不是global而是per request的。

`proxy_buffering` 是为了开启response buffering of the proxied server，开启后proxy_buffers和proxy_busy_buffers_size参数才会起作用。
无论proxy_buffering是否开启，proxy_buffer_size（main buffer）都是工作的，proxy_buffer_size所设置的buffer_size的作用是用来存储upstream端response的header。

在proxy_buffering 开启的情况下，Nginx将会尽可能的读取所有的upstream端传输的数据到buffer，直到proxy_buffers设置的所有buffer们被写满或者数据被读取完(EOF)。此时nginx开始向客户端传输数据，会同时传输这一整串buffer们。同时如果response的内容很大的话，Nginx会接收并把他们写入到temp_file里去。大小由proxy_max_temp_file_size控制。如果busy的buffer传输完了会从temp_file里面接着读数据，直到传输完毕。

一旦proxy_buffers设置的buffer被写入，直到buffer里面的数据被完整的传输完（传输到客户端），这个buffer将会一直处在busy状态，我们不能对这个buffer进行任何别的操作。所有处在busy状态的buffer size加起来不能超过proxy_busy_buffers_size，所以proxy_busy_buffers_size是用来控制同时传输到客户端的buffer数量的。

 

## 参考

原文：<https://www.cnblogs.com/xiewenming/p/8023090.html>

nginx官方模块说明： http://nginx.org/en/docs/http/ngx_http_proxy_module.html

淘宝的中文参考版： http://tengine.taobao.org/nginx_docs/cn/docs/http/ngx_http_proxy_module.html#proxy_buffering

其他参考： http://netexr.blog.51cto.com/2480285/1252245 http://blog.jiangming7.cn/toolssystem/server/nginx/692.html