### 配置详解

```bash
# redis关于内存分配使用的单位，不区分大小写，K、M、G单位进制1000，kb，mb，gb进制1024
##################################关于include指令 ###################################
# 一般的一台服务器上会跑多个Redis实例，而这些实例配置大同小异，因此将共同的部分抽取出来template.conf，
#然后在其他配置(Instance.conf)中引入是一种合理的实践方式，Redis配置支持这种实践模式,通过include指令；
#除此以外,在Instance.conf 覆盖参数也是合理的需求。
#Redis include处理的逻辑可以这么理解：
#Redis 利用Map 存储参数，按照定义的顺序读取及解析文件并肩参数存储到Map中，也就是说同样的配置靠后的才是生效的配置。
#最常用的实践方式为：在instance.conf文件的开头引入template.conf,并在instance.conf重写特定的配置
# include .\path\to\local.conf
################################## 以下为Redis使用的全部参数 #####################################
################################## 网络参数 #####################################
# bind 参数用于设定监听的本机IP（v4或者v6），可以是一个也可以是多个，如下所示：
# bind 192.168.1.100 10.0.0.1
# bind 127.0.0.1 ::1
# 如果bind参数没有定义, redis将会监听服务器的所有的IP（一般服务器有多个网卡）；
# 警告：在生产环境中（或者说在公网）监听全部的IP是极度危险的，所以默认的，Redis设定之间听回环IP(127.0.0.1),这样Redis只会处理来自本机的连接。
# 如果 你确认需要监听本机全部的IP，请注释掉bind 配置
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
bind 127.0.0.1
#protected-mode 用于设定是否开启保护模式，默认开启。
# 当Redis暴露在互联网，没有设定具体的监听IP，也没有开启访问的密码验证：
# 如果开启保护模式，那么Redis只会接受来自本机（本机回环地址）以及Unix Domain Socket（也是本机）的连接，
#关于 Unix Domain Socket ，请参看https://blog.csdn.net/guxch/article/details/7041052；
#基本类似于Socket 但是只用于本机进程间通讯，数据并不经过网络协议栈，没有打包解包流程，同时发送数据时会携带组Id，用户Id，进程Id用于接收方进程验证。
# 请确认好再做修改
protected-mode yes
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
#Redis监听的TCP端口 ，用于客户端连接，必须设定为大于0，且要避开系统端口
port 6379
#tcp-backlog 
#在高并发的环境下,需要提高tcp-bakclog 的值，用于保证连接较慢的客户端不会失去连接
#tcp backlog参数的作用 参看https://www.cnblogs.com/Orgliny/p/5780796.html
#基本上就是用于指定Redis可同时维持的最大的通信进程数，但是这个进程数不能超过Linux系统的默认设定值
#Linux 系统设定文件为/proc/sys/net/core/somaxconn somaxconn &cp_max_syn_backlog
#一般不需要设定
tcp-backlog 511

# Unix socket.
# 设定本机进程间通讯端口，默认不监听，一般也不用；
# unixsocket /tmp/redis.sock
# unixsocketperm 700

#设定Redis与客户端断开连接后与Redis关闭这个连接的时间差，0代表No
timeout 0

# TCP keepalive.单位second 一般60秒
# 用于设定Redis向客户端发出心跳检测的间隔时间，目的有两个：一个是心跳检测，一个是保证网络畅通（网络中间设备可能会掐断链路
tcp-keepalive 0
################################# 通用设置 #####################################
# daemonize 设定是否在以守护进程的模式运行，默认No，当开启后会在写一个/var/run/redis.pid 记录pid
daemonize no
# 如果使用 upstart or systemd 做服务的监控，可以配置这个选项用于发送信号给两者
# 可用选项：
#   supervised no      - no supervision interaction
#   supervised upstart - signal upstart by putting Redis into SIGSTOP mode
#   supervised systemd - signal systemd by writing READY=1 to $NOTIFY_SOCKET
#   supervised auto    - detect upstart or systemd method based on
#   UPSTART_JOB or NOTIFY_SOCKET environment variables
# 注意: 不保证有效，仅仅是发送信号给两者.
supervised no
# pidfile 用于设定Redis run on deamon 时 pid写到哪个文件
pidfile /var/run/redis.pid
# 设定Redis 日志的级别 
loglevel notice
# 设定日输出文件名，不设定日志输出会被重定向到标准输出流
logfile ""
# 设定 redis 分区数量，一般在多个应用使用一个群组的时候需要为不同应用分配不同空间避免Key冲突
databases 16

################################ 快照（数据定时写入磁盘）################################
# Save the DB on disk:
# 设定写入到磁盘的时机，要同时满足两条件 时间（单位seconds）、key的更改次数
# 如果不需要写到磁盘 用save "" 覆写
save 900 1
save 300 10
save 60 10000
# 如果开启了快照而最新的快照又失败的话,Redis将拒绝写入操作,以避免灾难发生,尽管这是很硬核的策略
#如果Redis 后台写快照又正常了，Redis则会自动的允许写入
#如果你设定正确的监控和持久化策略。你可以关闭这个特性(use no),即使硬盘坏了或者写出权限不足
stop-writes-on-bgsave-error yes
# 在Redis 把内存中的数据dump到硬盘上的rdb文件时，是否使用LZF算法对string 进行压缩，开启压缩的话，文件会变小，但是CPU小号功率会上升，默认开启。 
rdbcompression yes
# 从第5个发布的版本开始，Redis增加了在RDB文件的介未增加了CRC64的完整性校验位，防止文件被破坏。但是在存储的时候会降低大约10%的效率。默认yes开启no关闭
rdbchecksum yes
#设定mem数据dump到硬盘时的文件名 （Snapshot）
dbfilename dump.rdb
# 设定文件的存储目录 绝对路径或者相对路径均可以 只可以是文件夹：dafilename文件会存储在这个目录下
dir ./

################################# 主从复制设定 #################################
# 为了保证应用的的高可用（最高目标就是7*24*100%尽管实际上很难达到)。一般采用的套路有两种
# 一种是单机但是提供一个监控服务在应用服务程序挂掉马上重启（时间有间断用户可感知）；
# 一种是跑两个实例一台主机提供服务一台做备机备灾，主机挂掉，备机自动马上开始提供服务（无间断服务用户无法感知）。
# Redis 高可用是同通过双机切换来做的。
#具体的配置主机Mater和备机Slave关系可以使用如下参数（slaveof <masterip> <masterport>）
# Redis 主从的数据复制是异步的，Master可以配置备机的最小数量N，当与Master连接备机小于N时，Master将不会接受写入请求{Slave 数据和Master的数据对程序的表现是一致的（实际上并不一致，当Mater上有已过期的未被回收的Key存在时，这些Key不会复制到Slave上去，这个是Redis的一个Feature not Bug）}，一般情况下就是一主一备。
# 也可以i可以设定备机与主机的最长失联时间，在这个时间内连接恢复的话，同步会再次自动的进行不需要人工干预。
# slaveof <masterip> <masterport>

# 如果主机开启了password保护（requirepass 配置），需要使用下面的配置主机的访问密码
# masterauth <master-password>

#Slave，在失去了Master的链接或者在进行主从复制的时，收到客户端的请求时,
# 如果slave-serve-stale-data 配置为yes，Slave会继续响应这个读取请求（但是响应数据有可能为过期数据，或者空数据如果Slave是第一次同步数据）
#如果配置为no，Slave会响应给客户端一个ERROR(SYNC with master in progress)
slave-serve-stale-data yes
#我们可以配置一个Slave是否响应写请求，接受写请求可能对写如一些临时数据时有效果（会在主机数据进行同步时删除掉）
#Attention：但是很容易造成数据或者业务的混乱之类的问题，
#从Redis 2.6 开始 Slave默认为read-only，而且只读Slave并不适合暴露在互联网上,只读只是一个防止RedisSlave被错误使用的机制。
#除此之外，只读Slave仍旧可以接受一些管理命令比如config，debug等等。
#如果你想提高Slave的安全性，请使用“rename-command”命令对redis命令重命名,屏蔽掉管理\危险的命令,尽管这很有限。
slave-read-only yes

#数据同步的策略，通过硬盘或者socket
#Attention:不通过磁盘机型数据同步仅仅是试验中的特性；
#新增加的或者断线重连(如果不能继续原先的同步)的Slave会从Master进行全量同步（rdb文件被传输给Slave，Slave解析RDB文件）；
#传输的方式分为两种：
# 1.Master创建一个新的进程把数据不断写到硬盘上，然后再把文件传输给Slaves；
# 2.Master创建一个新的进程通过socket直接把rdb文件写到Slave上，不在经过硬盘；
#
#使用硬盘进行同步时,rdb文件生成完成后,Master再接受同步请求后就会把rdb文件发送过去；
#通过socket同步时,新的同步请求会被阻塞直到当前处理的同步请求完成；
#在使用socket同步时,Master可以配置一个等待时间以便于同时处理几个同步请求。
#如果网络带宽够大而且硬盘读写速度较慢时，socket是一个较号的选择。
repl-diskless-sync no
# socket同步请求处理延时，单位second
repl-diskless-sync-delay 5

#Slave 按照如下配置的时间ping Master，默认10seconds。
# repl-ping-slave-period 10

# 以下配置设定主从复制的相关超时设置：如果超时就会断开连接重连Master并且重新进行同步操作
# IO传输超时，从Slave的角度；（Slave发送SYNC到Mater,Master 需要在60s内把rdb文件内数据发送到Slave）
# Master网络超时，从Slave的角度；（Slave ping Master）
# Slave网络超时，从Master角度；（Master ping Slave）
# 注意这个值需要比repl-ping-slave-period 这个配置要大,不让当传输较慢的时候 总会触发超时。
# repl-timeout 60

# 完成数据同步后，Slave是否禁用TCP-NODELAY？
# 如果yes，Redis Master会将多份数据打包成一个小的数据报后再发送到Slave,会导致数据主从 数据上的延迟约40ms
#打包操作由linux内核完成，约延迟40ms，参看https://www.cnblogs.com/wajika/p/6573028.html
#if no,数据传输延迟会降低，但是带宽消耗会上升,默认会禁用延时。如果数据量很大或者主备机延时很高的情况设置为yes可能会好点
repl-disable-tcp-nodelay no

#设定主从复制的数据缓冲区大小，当Slave再复制期间断线，后又再次重连的话，如果数据缓冲区还未写满，Slave则会继续从缓冲区读入数据，否则会再次发送全量同步的请求。
# 只有存在Slave时，Redis才会申请这块内存
# repl-backlog-size 1mb

# 设定Master没有Slave connect多上时间后开始释放 backlog 内存
# repl-backlog-ttl 3600

# Slave的优先级（必须是正整数），决定了成为Master的顺位（当Master挂了的时候）。
#数值越小，优先级越高。但是如果是0的时候，则永远不会成为Master
slave-priority 100

#当在线的Slave小于N时，Master将不再接受写入请求。
#当Master收到Slave的最后一次ping 如果持续M秒时还未收到Ping，Master会Mark Slave offline
# 这配置并不能保证其他的N-1个Slave会接受写操作，只能保证少于N个Slave时Master不在接受写操作
# 0代表disable，默认的
# min-slaves-to-write 0， min-slaves-max-lag 10
# min-slaves-to-write 3
# min-slaves-max-lag 10
################################## 安全保证###################################
#是否需要开启密码验证（一般情况下，并不需要）,Redis 性能相当高，大约每秒处理大约150k请求，因此密码最好很长，不然直接就暴力破解了
requirepass foobared

# 修改命令的名字为新的字符，比如 rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
# 这样的话 客户端只能用过 b840fc02d524045429941cc15f59e41cb7be6c52 来达到 config的的效果，当rename-command CONFIG “” 时，CONFIG命令就会失效
# 注意：如果修改的命令会被记录到AOF或者传输到Slave机器会造成很多问题。
# Command renaming.
# Please note that changing the name of commands that are logged into the
# AOF file or transmitted to slaves may cause problems.

################################### 限制 ####################################
#设定Redis同时最大的客户端连接数量，默认情况为10000，如果Redis无法从配置文件进行配置，Redis将会设定为当前文件的限制减去32；
maxclients 10000
# 是否开启持久化机制，如果否，那么一切持久化相关配置都会失效。比如RDB（内存数据快照）&AOF(操作记录AppendOnlyFile)
# persistence-available [(yes)|no]

#当内存使用量达到设定的上限时，Redis就会开始回收过期的Key.
#如果无法删除key,Redis会对使用大量内存的命令（lpush等）返回错误信息，但是GET类命令正常响应。
#除此之外,如果当前Redis有备机存在，那么写到备机时所可使用的最大缓存大小为机器内存减去Redis数据所占内存。
#因此,当数据过多时就会挤压主从缓冲的大小，直至缓冲被挤占完全，当网络有问题时，会造成同步失败
#简单的说，当有备机存在时，maxmemory需要设置的小点给主从复制缓冲留些空间
#注意：这是为0时,代表不限制内存的使用，当内存用尽时会OOM
#Redis使用系统内存分页申请堆,其他的内存占用分析工具查看Redis使用的内存大小时并不准确
# maxmemory <bytes>

#内存用尽时的Key删除策略，共5种
# volatile-lru -> 删除 Least Recent Use 算法& Has Expried Keys
# allkeys-lru ->删除 Least Recent Use 算法 & Any  Keys
# volatile-random -> 随机删除A  Has Expried  Key 
# allkeys-random -> 随机删除A Key
# volatile-ttl -> 删除距离过期时间最近的Key
# noeviction -> 不做任何处理,当达到maxmemory时,直接return an error to client

# maxmemory-policy noeviction

# LRU或者TTL并不是精确算法,可以对算法进行调整，默认的，LRU或者TTL 每次会选择5个Key检测,随机的删除满足条件一个。不需要修改，5 is tested for the best performance
# maxmemory-samples 5

##############################只追加模式  ###############################
# 默认情况下,Redis dump data to disk 和接受读写的线程是异步的，这种模式在大部分情况下足够用了，但是如果突然停电关机，一般情况下，Mem内的数据 会丢失一部分（不确定的）。
# AOF 模式提供了更可靠的持久化方式，使用fsync 策略时，Redis只会丢失1s的写入数据(取决于appendfsync 配置)。
#AOF 和RDB 可以同时启用，但是会优先使用AOF
appendonly no
# aof 文件名
appendfilename "appendonly.aof"
# Redsi调用fsync()时，会告诉系统立刻把数据的从内存写入到 硬盘而不是停留在在输出buffer里面。（但是取决于系统，linux下是如此）
# Redis 调用 fsync()模式有如下几种：
# no，不调用，让系统决定；
# always 每一次写操做都会调用
# everysec每秒调用一次，默认值

# appendfsync always
appendfsync everysec
# appendfsync no

#当AOF fsync 为always 或者 everysec，同时在后台有进程在进行大量的磁盘IO（比如BGSAVE或者AOF重写时），
#在一些Linux配置中Redis会长时间的阻塞在fsync调用上（目前还没有办法解决），甚至至其他线程调用也会阻塞我们对write(2)的调用(因为Redis Main Thread调用write是同步阻塞的）
#为了减少这个问题请使用如下的配置，以阻止主线程对fsync的调用（当有BGSAVE或者BGREWRITEAOF执行的时候）
# 这样有一个后果，AOF文件从内存写入到硬盘完全由系统控制即appendfsync 为no的效果一样，最坏的情况下，AOF会丢失30s的数据
#如果你对读写效率要求非常高时，请将下面的配置为yes，不然请保持no，这样是数据最安全的
no-appendfsync-on-rewrite no

#AOF 持久化是通过保存被执行的写命令来记录数据库状态的，所以AOF文件的大小随着时间的流逝一定会越来越大；
#影响包括但不限于：对于Redis服务器，计算机的存储压力；AOF还原出数据库状态的时间增加；
# 为了解决AOF文件体积膨胀的问题，Redis提供了AOF重写功能：
# Redis服务器可以创建一个新的AOF文件来替代现有的AOF文件，新旧两个文件所保存的数据库状态是相同的，
# 但是新的AOF文件不会包含任何浪费空间的冗余命令，通常体积会较旧AOF文件小很多。（主要是合并一些命令）
# 这个机制触发机制是：当AOF文件大小的增长到某一个值时；
# auto-aof-rewrite-min-size 配置 重写的触发时最小的AOF大小，
#auto-aof-rewrite-percentage 配置当AOF文件相比于Redis启动时大小增长的比率，0代表禁用BGAOFREWRITE特性
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

#在Redis启动时，AOF文件被载入内存，有可能持续出现（AOF被截断文件不完整这种情况，一般是系统挂掉造成的尤其是EXT4这种文件系统），Redis根据如下的aof-load-truncated的配置要么直接报错退出，要么尽可能的载入可载入的数据。
#aof-load-truncated yes 时，Redis会正常启动并尽可能的载入数据，并将AOF损坏报告给用户
#aof-load-truncated no时，Redis将会因为报错而退出，需要使用redis-check-aof来修复AOF文件后再次启动。
# 注意，这个选项支队啊、AOF文件结尾损坏有效，如果AOF文件中间损坏了并不会由任何效果Redis人仍旧会退出报错。
aof-load-truncated yes

################################ LUA 脚本  ###############################
#配置一个Lua脚本执行时，最大的毫秒数，如果超过这个时间脚本仍在执行，Redis将会记录这个情况,并且会对所有的读取请求回复错误。
#当脚本执行时间超标了，那么只有SCRIPT KILL and SHUTDOWN NOSAVE可以使用；
#SCRIPT KILL 只能应用在脚本还未执行写操作时，
#SHUTDOWN NOSAVE 适用于脚本有了写操作，而用户又不想等待脚本自然终止的情况下；
#设置为0 或者负数时，代表不设置最高时间上限。
lua-time-limit 5000

################################ REDIS 集群###############################
#警告：Redis集群尽管被认为是个稳定的版本，但是仍需要一些吃螃蟹的人在生产环境中大规模使用才能被标记为成熟。
# 一般的正常的Redis时无法成为集群的一部分的，只有以集群节点方式启动的Redis才可以加入集群。
# 将如下配置为yes，Redis启动模式将会改为集群节点模式；
# cluster-enabled yes

#每个Redis节点都有一个自己的集群配置文件，但是这个文件不适于手动编辑，而是由Redis节点生成修改；
#在单机配置 多个节点的时候，请确认每个节点都有自己的配置，不会相互复写；
# cluster-config-file nodes-6379.conf

#集群超时时间是这个节点不可达的最长毫秒数，不然节点见会被标记为failure，一般是redis内部超时的整数倍
# cluster-node-timeout 15000

#集群主机的备机，将不会自动的切换为主机状态，如果 备机数据too old，
#检测备机数据的年龄，Redis依靠如下两个检测结果：
# 1）如果存在多个备机可以切换为主机,那么备机将相互交换数据查看谁备份数据最好(接受处理的Master发送的同步数据最多)
# 2）每个备机将会计算自己与Master最后一次通信（互ping，接受命令）的时间，或者与Master失联的时间，如果时间太大则根本不会切换为主机，
# 判断标准为{(node-timeout * slave-validity-factor) + repl-ping-slave-period}单位秒。
# 当设置slave-validity-factor 为0 时，Slave会忽略数据年龄判断，加入Master选举
# cluster-slave-validity-factor 10

# 集群备机能够迁移成为其他单点主机的备机，这样可以提高集群的稳定性；
#但是备机迁移的前提是老的主机备机数量大于老主机指定的最小备机数；
#默认情况下，每个主机只需要要有一个备机。
#但是为了节点的稳定性要避免备机迁移，请将数值调很大，
#也可以设置为0，但是只是请旨在debug情境下使用，生产环境会很危险

# cluster-migration-barrier 1

#默认情况下，只要有一个slot不可使用，整个集群立刻进入不可用状态（不可写也不可读），当这个slot可用了整个集群又会回复；
# 如果你想使仍在工作的部分机器继续接受读取请求，请将下面的配置设为no
#备注：slot 是每个Key存储的位置,Redis设计有16384 slot，
#对于任何一个的Key的任何操作都需要确定他的在哪个slot（通过CRC64计算Key的hash，然后对16384求余即可得到slot id）
# cluster-require-full-coverage yes

################################## SLOW LOG ###################################
#SLOW LOG 用于记录执行超时的命令,(执行时间的计算不包括与IO时间仅仅是Redis执行命令的时间，这个时间内Redis仅只执行这一个命令，命令处理是单线程的)，
# 可以配置最大执行时间（单位ms）及记录超时命令的队列长度（队列满后老命令会被挤出）
slowlog-log-slower-than 10000
slowlog-max-len 128

################################ LATENCY MONITOR ##############################
#LATENCY MONITOR 主要是Redis集群用来收集子节点有哪些命令执行超时（>= latency-monitor-threshold的 值),通过LATENCY命令可以获取可视化的数据及报告
#默认地，这个系统是禁用的（设置为0），如果想在运行时启用可以使用“CONFIG SET latency-monitor-threshold <milliseconds>”命令
latency-monitor-threshold 0

############################# EVENT NOTIFICATION ##############################
#Redis 支持Pub/Sub 模式，Redis可以通知客户端发生了什么
#可以设定Redis需要对哪一类事件进行通知，
#  K     Keyspace events, published with __keyspace@<db>__ prefix.
#  E     Keyevent events, published with __keyevent@<db>__ prefix.
#  g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
#  $     String commands
#  l     List commands
#  s     Set commands
#  h     Hash commands
#  z     Sorted set commands
#  x     Expired events (events generated every time a key expires)
#  e     Evicted events (events generated when a key is evicted for maxmemory)
#  A     Alias for g$lshzxe, so that the "AKE" string means all the events.
#notify-keyspace-events 支持多个类型事件直接拼接即可，默认情况下会禁用
notify-keyspace-events ""

############################### ADVANCED CONFIG ###############################
#存储的的Value REDIS_LIST,list.size<64 ,最大element长度不超过512 ，那么这个 RedisList 会被重新Hash以节约内存
#否则使用 HashTable来存储
hash-max-ziplist-entries 512
hash-max-ziplist-value 64

#RedisList 数据会被重新编码以节约内存,可以指定每个Node的内存大小或者可以持有的元素数量
#负数标识内存大小
# -5: max size: 64 Kb  <-- not recommended for normal workloads
# -4: max size: 32 Kb  <-- not recommended
# -3: max size: 16 Kb  <-- probably not recommended
# -2: max size: 8 Kb   <-- good
# -1: max size: 4 Kb   <-- good
#整数表示可有存储的元素数量
#一般每个node 8kb足够好了 不需要修改
list-max-ziplist-size -2

#Redis List 也可以压缩，可以设定压缩的深度，即前多少个节点可以压缩默认0不要要锁
list-compress-depth 0

#Sets 只有满足如下情况是才会被压缩：
# 这个set 完全由64bit所能表示的有符号整数组成时；
# 如下条目配置了一个Set最大元素数量
set-max-intset-entries 512

#和Hash和List相似，有序Set也会被重新压缩编码节约内存，当仅当Set内的元素数量小于64.元素bit数小于128时；
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

#基数统计分组大小（主要用于统计Keyde数量通过HyperLogLog 算法，别去看：涉及数理统计及数据分布，数据可信度等等内容）
hll-sparse-max-bytes 3000

# 激活ReHash Redis Main hash table（存储Top key的那个，指SET a 命令存储 a 的hash table） 大约只占用1%的CPU时间片。
# Redis 对主表的rehash延迟执行，对正在HashTable操作越多，执行rehash的步骤越多,因此处于空闲的Redis，rehash永远无法完成
#默认情况下每秒执行10次rehash操作用于释放内存；
# 如果不确定或者对redis读取有很高的速度要求,请将activerehashing  设置为 no，
# 如果尽快释放内存，请设置yes
activerehashing yes

# 客户端输出缓冲区限制可以被用来青珀客户端断开与服务器的链接（如果客户端读取很慢而服务器生成很快情况下）
# 可以对三种不同类型的客户端normal、slave、pubsub）进行限制，限制的语法为（注意0为禁用）
# client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
# 当缓冲区占用了大小达到 hard limit 时，或者 达到 soft limit 达到 soft secnds 后，客户端会马上被强制断开链接

# 默认情况下，normal 不会别限制，因为只有在异步的pubsub模式下才会出现输出缓冲被占满的情况；
#其他的情况，有默认的限制
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# Redis 通过调内部函数执行很多后台任务（关闭超时连接,废Key回收等等），尽管这些任务执行的频率并不一样，但是我们可以设定检测需要执行任务的频率（hz 10 范围 1-500，但一般不要超过100除非需要很低的垃圾回收延迟）;
#Redis 空闲时会略微提升CPU功率，但是同时会提升垃圾回收的速度，
hz 10

#在后台重写AOF的时候，如果开启aof-rewrite-incremental-fsync yes，重写的AOF将会没32MB存到DISK一次，以避免IO速率出现尖峰形态
aof-rewrite-incremental-fsync yes

################################## INCLUDES ###################################
# include /path/to/local.conf
# include /path/to/other.conf
```

