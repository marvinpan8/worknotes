# Redis服务端优化实践：配置优化、主从切换、持久化

[文章链接](https://mp.weixin.qq.com/s?__biz=MzI4NTA1MDEwNg==&mid=2650764050&idx=1&sn=891287b9f99a8c1dd4ce9e1805646741&chksm=f3f9c687c48e4f91c6631e7f5e36a9169c10549386bec541dbeef92ed0023a373f6ec25c2ef1&mpshare=1&scene=21&srcid=0525xnHQxiFwpzFWSME2LQrb#wechat_redirect) **闫东旭**，网易乐得高级工程师。



Redis是部门业务重要的核心业务组件，被广泛应用在行情系统、推送服务、数据中心、投顾、圈子、量化分析等平台。在使用Redis的过程中遇到了很多问题，涉及到开发者使用API时的一些注意事项，以及如何通过优化服务端配置提高Redis的健壮性、容错性。

本文通过案例分析的方式分享一下我们在Redis服务端配置优化、主从切换、持久化等方向的实践。

注：Redis Server端版本2.8.19，客户端使用Jedis2.6.0。

## 一、Server配置

**allkeys-lru还是noeviction**

Redis的服务端有这样一个配置参数maxmemory-policy allkeys-lru，是说Redis的使用内存达到分配上限的时候默认使用lru的key淘汰策略（还有其它可选策略）。我们曾经遇到这样一个问题，应用端因为一次不当的hgetall操作，导致Redis使用内存膨胀超过上限，触发了key淘汰，结果重要数据被清空（幸好大部分业务都做了灾备恢复机制），图中可见使用内存上升后又迅速回落，是因为触发了淘汰算法。


![img](http://mmbiz.qpic.cn/mmbiz_png/tibrg3AoIJTuvA1Ts2M3pOyv3JwqMHKw7mV8MNLkxsrfqE65KRurFNynRdUM7Kjr3TgndOaLMIibdgosBwKVCxWg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这次事故之后我们将淘汰策略改为noeviction—禁止key淘汰。那么，到底要不要调整淘汰策略呢？其实主要看业务场景，Redis是被当作允许数据淘汰的缓存还是存储核心数据的内存数据库。

使用Redis的一个核心优势就是Redis具有丰富的数据结构，当前的大部分业务也都是看中这一点才选择Redis，某些复杂结构的数据集一旦被清空，没有业务上的恢复机制是不可能自动重建的。如果你的Redis不是简单的字符串缓存，那么就需要慎重考虑是否禁用key淘汰了。

```properties
client-output-buffer-limit normal 0 0 0 ？
```

美团曾经因为工程师开启monitor命令导致内存飙升引发事故，我们hgetall的例子与之同出一辙，原理都是因为Redis在处理客户端请求的时候会为每个请求分配一个输出缓冲区，如下图：


![img](http://mmbiz.qpic.cn/mmbiz_png/tibrg3AoIJTuvA1Ts2M3pOyv3JwqMHKw732oDS2icSib3otCUdgSxFVTFbl2klFQ8TYOA0ck0TBnBsN7uor4c0j2Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



默认这个缓冲区是无限分配的，而且同样占用Redis的内存空间，也就意味着一个大查询过来使用内存就可能成倍飙升，多查几次内存就爆了，如果不幸又没有设置禁止key淘汰，那么数据很可能就被清0了。Redis的API提供了类似hgetall、smembers、keys等很多O(N)级别指令，如果你的应用QPS很高，或直接面向终端用户请求，那么一定慎用O(N)级别指令；如果指令的目标数据集很大，那么意味着要么的请求很耗时会长期占用cpu，要么查询的数据量很大导致Redis内存飙升。Redis单线程的设计注定不是为高频大数据集查询准备的。

我们对这个参数设置了client-output-buffer-limit normal 10mb 5mb 10 的限制，单个请求10mb缓冲区的分配已经足够高了，还可以更低，具体视应用而定，但绝不建议无限分配。

```properties
lua-time-limit
```

Redis支持自定义命令的一个优势是因为支持Lua脚本，我们部分业务使用了Lua脚本，这些自定义指令的运行时间需要被测试、评估，在要求QPS比较高的应用中，一定不要有Lua脚本长期占用CPU资源。虽然lua-time-limit不会终止Lua脚本，但会使Redis到达时限后开始响应客户端请求返回Busy错误，这样能在大量连接被挂起、超时时，避免不知道发生了什么。Lua时限需要设定，但还是建议提前测试lua脚本的性能。

```properties
rename-command
```

Redis提供了rename-command可以改写某些危险命令，使其无法执行成功，这里建议可以禁掉如下几个命令：

- rename-command FLUSHALL “”  /*删除所有现有的数据库*/
- rename-command FLUSHDB “”  /*清空当前数据库中的所有key*/
- rename-command SHUTDOWN “”  /*关闭 redis 服务器(server)*/
- rename-command KEYS “”  /*复杂度O(N)，遍历所有key*/
- rename-command MONITOR “”  /*debug命令，查看Redis正在执行的命令*/

公司曾有工程师误执行过flushall导致数据清空，美团掉过monitor的坑，我们也出现过使用keys造成的CPU卡顿。这些命令其实不必也不应该出现在应用程序API调用中。寄希望于工程师不要误调用，不如在server端直接禁掉。

```properties
slaveof
```

这个指令很特别，它指定了当前Redis实例是某个Master实例的slave。这个指令如果在配置文件中写死，那么实例启动后就只能是slave，除非有哨兵将它提升为master，或手动执行slaveof no one。这个指令是一个会被哨兵动态从配置文件里删除或者添加的指令，它的存在与否最好交由哨兵决定。

我们之前的配置文件是通过子文件引入的这个指令，这样会有一个问题，如果某个slave被哨兵选举成为了master，它的slaveof指令要被移除，但子文件中写死的指令是无法移除的，一旦重启这个master实例，它又成了slave。如果不注意这个子文件的存在，问题还很不好排查，不知道发生了什么。线上没有遇到过问题，测试环境做过一次主从切换，操作过程中，重启了下master，结果掉坑了。这个测试的例子后面还会讲，因为它还涉及到哨兵及主从切换。

## 二、主从切换

Redis的使用一般有单点和集群两种方式，对QPS有很高的要求一般会搭建集群，目前我们这边一般业务对QPS没有那么高的需求，所以都是**单点主从模式（集群各节点也是主从模式**）。

说到主从切换不得不先说哨兵，Master节点故障时势必要求能及时地切换到slave节点，尽快恢复应用。那么监测Master状态，发起主从切换过程，最终将新选举的master通知给客户端应用的任务就是由哨兵完成的。哨兵一般需要部署一个集群。


回到上面的例子，讲这个例子是因为后面的调研及对配置的调整都是由这个例子引起的，故事是这样的：

1. 第一次主从切换，关闭r1，master顺利切换到r2
2. 启动r1，主从同步后r1被误关闭
3. 第二次主从切换，关闭r2，切换失败（r1被误关闭，尴尬）
   （这时候的配置是这样的：
   r1在之前第2步启动后被哨兵打上了slaveof r2 6379标记；
   r2主配置文件中哨兵会移除slaveof标记，但子文件中的标记无法移除；
   因为切换失败，哨兵的配置文件中master的配置还是r2；）
4. 切换失败了，重启r1，因为本来就要切到r1嘛，结果哨兵没反应，应用端还是报错，看日志应用端还是在试图连接r2。
5. 没办法，重启r2，结果应用端换了个报错，slave模式不可写。也就是r2连通了，但slave模式不可写数据（slave模式可写与否可以配置，我们这里配置slave只读）。
6. 这时候用info命令查看r2的角色的确是slave，查看r1也是slave。

这里就蒙圈了，r1和r2都起来了，却没有master，master哪里去了，大概持续了10分钟，奇迹也没有发生，**对r2强制用slaveof no one命令提升为master，服务恢复**。整个过程走下来有很多疑问，之前没有了解过Redis主从切换的知识，带着疑问开始在本地模拟测试，这里主要有三个问题：

1. 为什么重启后两个实例都是slave
2. 为什么哨兵没有从两个实例中选择一个成为master，并通知客户端。
3. 主从切换这么重要的过程什么情况下会失败呢？

**第一个问题，**上面已经讲过了，重启后两个实例都是slave，这是因为第一次切换r1成为master，r1重启会被哨兵打上slaveof标记，被误关闭当然slaveof标记是不会清除的了。r2虽然是master，但重启的时候子文件中指定了slaveof，哨兵无法移除，所以重启也就进入了slave模式。

**第二个问题，**哨兵为什么没有发挥作用？其实如果我们再等一段时间，奇迹是可以发生的，等多久呢，总共需要等30分钟（是不是有点太久了？）。Sentinel有一个重要的配置参数**sentinel failover-timeout master 900000**，**这里我们默认的配置是900000ms即15分钟**，如果sentinel操作过一次故障转移但失败了，那么下次故障转移的时间是2*15=30分钟。上面的例子当关闭r2的时候哨兵已经做过一次切换了，但r1也挂了，所以这个failover是失败的。需要等30分钟才能做第二次，重启r1和r2，双slave困局也就没人来破解了。由于等待时间太长了，不知道发生了什么，让人以为哨兵是不是挂了。官方默认配置是180000，我们因为历史原因提高了这个时限，恢复默认配置足够了。

**第三个问题，**什么情况下主从切换会失败？调研了部分资料，并本地模拟我们的当前配置搭建了Redis环境做了一系列测试，发现以下问题：

**（1）超过一半以上的sentinel宕机，剩余的sentinel无法进行failover**

​	这种情况如果只有两个sentinel，且其中一个sentinel和master实例同机部署，如果因为虚拟机故障或网络问题master被认为宕机，同机的sentinel基本也就挂了，不能切换的概率100%。sentinel选举leader需要超过一半的sentinel同意，少数的sentinel永远无法选举出leader，所以sentinel最好不要只设置两个，而且sentinel最好不要与redis实例同物理机部署，否则redis实例宕机很大可能同机的sentinel也挂了，sentinel就失去了意义。

**（2）大部分的sentinel投票给自己，没有candidate获取超过一半的票数，所以没有选举出leader，需要等待2*failover-timeout时间，重新发起切换。**

​	这种情况在只有两个sentinel时发生的相当频繁，sentinel1和sentinel2同时观察master客观下线，同时发起投票，每个sentinel都选自己，无法选举出leader。多个sentinel理论上也有可能出现选举失败，概率低一些。尤其第一次选举失败，下次选举要等待2*failover-timeout，我们这里是30min，会让大多数人觉得不知道发生了什么。这里还是建议根据自己业务使用情况**，尽可能缩短failover间隔。**

**（3）即使sentinel成功选举出了leader，切换还是不一定成功，不是所有存活的slave都可以提升为master，需要经过筛选。**

- slave节点状态处于S_DOWN, O_DOWN, DISCONNECTED的不能通过筛选
- 最近一次ping应答时间不超过5倍ping的间隔（假如ping的间隔为1秒，则最近一次应答延迟不应超过5秒，redis sentinel默认为1秒），否则不能通过筛选
- info_refresh应答不超过3倍info_refresh的间隔（原理同2, redis sentinel默认为10秒）
- slave节点与master节点失去联系的时间不能超过指定阈值**10*down-after-milliseconds（我们一般设置60s，10倍就是10分钟）**，意思是说，与master同步太不及时的slave，不应该参与被选举。
- Slave priority不等于0（这个是在配置文件中指定，默认配置为100）。



http://ks.netease.com/blog?id=7468 网易实践者社区分享的这次事故集群中12个节点7个主从切换失败就是由于slave节点与master失连太久造成了没有成功切换，当然这次事故还有其它原因：

​	1）**master节点和同机部署的sentinel都出现了问题**

 	2）**3个sentinel挂掉了一个**

如果3个sentinel独立部署，这个故障很大程度能够避免。

### 应急预案（本文以Jedis2.6.0客户端为例）：

**由于主从切换可能失败，具体原因具体分析：**

1. 如果是因为大部分sentinel都挂了，不足以选举出leader，选择一个存活的sentinel**强制执行sentinel failover吧，**如果有备选slave成功被提升为master，客户端应用不需重启。
2. 如果是因为哨兵没有选举成功，看等待时间是否还能忍，能忍就等待下次failover，否则同1）步骤。
3. 如果是因为没有slave通过筛选，建议停止全部哨兵，将某个slave提升为master，修改哨兵配置指向这个新的master，重启哨兵，重启客户端应用（还可以再起一个Redis实例，新实例slaveof 新master， 手动在sentinel上做下failover，这样就不用重启客户端应用了~）

### 哨兵部署建议：

1. 按奇数个部署，**至少要部署3个，哨兵之间、Redis实例之间物理机独立。**

2. **sentinel monitor master xxx.xxx.xxx.xxx xxxx 1**

   哨兵的这个配置最好不要配置为1。quorum的值为1意味着只要一个sentinel发现master节点无响应就可以标记为客观下线，从而发起主从切换，quorum最好设置超过sentinel个数的一半向上取整。

3. **sentinel failover-timeout master 900000 //毫秒级**

   条件允许的情况下尽可能**缩短这个切换间隔吧。**



## 三、持久化

Redis有两种持久化方式，AOF和RDB，AOF持久化是指追加写命令到aof文件的方式，RDB是指定期保存内存快照到rdb文件的方式。

RDB虽然可以通过bgsave指令后台保存快照，但fork()子进程是有开销的，在内存数据集较大的情况下会占用很长的cpu时间，fork新进程时，虽然可共享的数据内容不需要复制，但会复制之前进程空间的内存页表，如果内存空间有40G（考虑每个页表条目消耗 8 个字节），那么页表大小就有80M，这个复制是需要时间的，在有的服务器结点上测试，35G的数据bgsave瞬间会阻塞200ms以上，一般建议Redis使用内存不超过20g。I/O消耗，我们线上是在Slave节点开启rdb持久化，磁盘性能一般，1.2g的rdb文件持久化一分钟一次，一次大概耗时30s左右，所以rdb的频率也不能太频繁，需要根据情况做好配置。

AOF是追加写命令到aof文件的方式，优点是可以基本做到数据无损，缺点是文件增长较快，需要间歇性bgrewrite，bgrewrite也是一个既耗cpu又耗磁盘IO的操作，单cpu利用率最高可达100%。bgrewrite期间可以设置将新的写请求暂时缓存，bgrewrite完成后同步写盘，同步会暂时停止处理客户端请求，如果bgrewrite时间较长，缓冲区积压数据较多，核心阻塞时间会很长，所以如果必须要开启aof，一般建议找几个空闲时段设置脚本来做bgrewrite。

AOF还有一个比较坑的地方是刷盘策略fsync的设置，这个设置一般有3种方式：always、everysec、no，如果设置为no，就将写盘的时机交给操作系统，这在很大程度上牺牲了aof数据无损的优势，如果设置为always就意味着每条命令都会同步刷盘，会造成频繁I/O，所以一般建议是设置everysec，Redis会默认每隔一秒进行一次fsync调用，将缓冲区中的数据写到磁盘。但是当这一次的fsync调用时长超过1秒时。Redis会采取延迟fsync的策略，再等一秒钟。也就是在两秒后再进行fsync，这一次的fsync就不管会执行多长时间都会进行。这时候由于在fsync时文件描述符会被阻塞，所以当前的写操作就会阻塞，因为是同步操作所以核心处理阻塞，开启aof且要求Redis性能无损对磁盘有极高要求。下图是我们一段时间内的磁盘监控截图：


![img](http://mmbiz.qpic.cn/mmbiz_png/tibrg3AoIJTuvA1Ts2M3pOyv3JwqMHKw79TZr0XL53e4FXgIFQSlKPuknyGH1FqRssMlBKffwoPeZKAv4ddWniaA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



这种间歇性的磁盘IO毛刺就会使fsync阻塞，fsync阻塞时一般会输出如下日志：


![img](http://mmbiz.qpic.cn/mmbiz_png/tibrg3AoIJTuvA1Ts2M3pOyv3JwqMHKw7YJEMMvDTo74tic3WqibgML3FzyhouuaUcLASPC0AkkGVu6QSADxbkhTQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



持久化为Redis提供了异常情况下的数据恢复机制，但开启持久化是有代价的，哪一种持久化都可能造成CPU卡顿，影响对客户端请求的处理。不开启持久化又存在风险，如果一旦误重启master节点，或者试想这样一种场景，主从切换失败，很可能因为疏忽直接重启master，这时没有开启持久化的master会把所有slave的数据清0。所以是否开启持久化，怎样开启持久化是一个难题。和运维同事探讨了一些方案，这里总结一下供大家参考：

1、极端情况下可以容忍全量数据丢失，那么建议master关闭持久化，slave关闭持久化；

2、极端情况下不能容忍全量数据丢失，但可以容忍部分数据丢失，如果内存数据集较小且不会增长建议master开启rdb，slave开启rdb；如果数据集很大，或不确定数据集增长趋势，建议master关闭持久化，slave开启rdb

开启rdb需要cpu和磁盘性能保障。如果master关闭持久化，slave开启rdb需要保证slave的rdb不会被master误重启所覆盖，这里提供几种方案：

- 重启脚本包一层命令先网络请求加载备机备份目录下的rdb文件后再执行start，可以防止误重启，但备机调整部署可能需要调整脚本，主机打开持久化也需要调整脚本
- 定时将rdb文件通过网络io传给master节点（文件大比较耗时，文件增长需要考虑定时脚本执行间隔，否则会造成持续的网络io），而且也会有一定数据损失
- 定时备份Slave的rdb到备份目录，不做任何其他操作，误重启时人工拷贝rdb到master节点（会有一定数据损失）

3、最大限度需要数据无损，建议master开启aof，slave开启aof

​	开启aof需要cpu和磁盘性能保障。开启aof建议fsync同步刷盘使用everysec，自定义脚本在应用空闲时定时做bgrewrite，bgrewrite期间增量数据做缓冲。

​	**目前大部分业务都允许部分数据丢失，为使Redis性能最大化，关闭了Master持久化，slave开启rdb，为防止误重启对rdb做了5分钟一次备份，保留最近1小时的备份文件，必要时人工copy到master数据目录下恢复数据。后续硬件性能提升后，看情况再调整持久化机制。**