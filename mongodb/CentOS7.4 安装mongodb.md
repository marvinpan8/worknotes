# CentOS7.4 安装mongodb

[ 参考文章](<https://www.jianshu.com/p/994bc7b19b26>)

温馨提示：我的环境是腾讯云自带的CentOS7.4 x64 镜像，本地环境是win10 x64 专业版，ssh工具是用的win10 自带的cmd, 远程工具版本是Robo 3T 1.2.1 。
 如果环境不一致，可能会出现无法预知的错误。

1、去官网找到安装包地址，复制下来。
 官网地址：[https://www.mongodb.com/download-center?jmp=nav#community](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.mongodb.com%2Fdownload-center%3Fjmp%3Dnav%23community)
 我使用的安装包地址：[https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-4.0.0.tgz](https://links.jianshu.com/go?to=https%3A%2F%2Ffastdl.mongodb.org%2Flinux%2Fmongodb-linux-x86_64-4.0.0.tgz)

2、使用SSH登录服务器，找一个文件夹存放安装包，我这里使用的是 /usr

```bash
$ cd /usr
$ wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-4.0.0.tgz
```

第一步是定位到/usr文件夹，第二步是下载安装包。



![img](https:////upload-images.jianshu.io/upload_images/2103305-4367461e3990e5df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/936/format/webp)



如图所示进度到100%时，就是下载完成了。

3、解压缩安装包，并重命名文件夹。

```bash
$ tar zxvf mongodb-linux-x86_64-4.0.0.tgz
$ mv mongodb-linux-x86_64-4.0.0 mongodb
```

第一步是解压缩，第二步是重命名，如图所示。



![img](https:////upload-images.jianshu.io/upload_images/2103305-9cab80ce464ad5d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/629/format/webp)

解压缩



![img](https:////upload-images.jianshu.io/upload_images/2103305-5a458bbfca75960a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/535/format/webp)

重命名

4、配置环境变量

```bash
$ vim /etc/profile
```

在 export PATH USER LOGNAME MAIL HOSTNAME HISTSIZE HISTCONTROL 一行的上面添加如下内容:

```bash
#Set Mongodb
export PATH=/usr/mongodb/bin:$PATH
```

保存后通过下面的命令使环境变量生效：

```bash
$ cd ~
$ source /etc/profile
```



![img](https:////upload-images.jianshu.io/upload_images/2103305-e6254a19c956fe7c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/627/format/webp)

设置环境变量

5、创建数据库目录

```bash
$ cd /usr/mongodb
$ touch mongodb.conf
$ mkdir db
$ mkdir log
$ cd log
$ touch mongodb.log
```

6、修改mongodb配置文件。

```bash
vim /usr/mongodb/mongodb.conf
```

添加以下内容

```properties
port=27017 #端口
dbpath= /usr/mongodb/db #数据库存文件存放目录
logpath= /usr/mongodb/log/mongodb.log #日志文件存放路径
logappend=true #使用追加的方式写日志
fork=true #以守护进程的方式运行，创建服务器进程
maxConns=100 #最大同时连接数
noauth=true #不启用验证
journal=true #每次写入会记录一条操作日志（通过journal可以重新构造出写入的数据）。
#即使宕机，启动时wiredtiger会先将数据恢复到最近一次的checkpoint点，然后重放后续的journal日志来恢复。
storageEngine=wiredTiger  #存储引擎有mmapv1、wiretiger、mongorocks
bind_ip = 0.0.0.0  #这样就可外部访问了，例如从win10中去连虚拟机中的MongoDB
pidfilepath=/jrtz/mongo/mongodb/mongod.pid
```

7、设置文件夹权限

```bash
$ cd /jrtz/mongo
$ chmon mongodb:mongodb mongodb
$ cd mongodb
$ chmod 755 db
$ chmod 755 log
$ chown mongodb:mongodb db
$ chown mongodb:mongodb log
```

8、启动mongodb

```bash
$ cd ~
$ mongod --config /usr/mongodb/mongodb.conf
网友指正：最新版本mongodb已经将--config 修改为 -f (本人尚未尝试)
```

9、开机自启动

```bash
$ useradd -m mongodb
$ cat << EOF | tee /etc/systemd/system/mongodb.service
[Unit]
Description=mongodb
After=network.target remote-fs.target nss-lookup.target

[Service]
User=mongodb
Type=forking
PIDFile=/jrtz/mongo/mongodb/mongod.pid
RuntimeDirectory=mongodb
RuntimeDirectoryMode=0751
ExecStart=/jrtz/mongo/mongodb/bin/mongod -f /jrtz/mongo/mongodb/mongodb.conf  
ExecStop=/jrtz/mongo/mongodb/bin/mongod --shutdown -f /jrtz/mongo/mongodb/mongodb.conf  
PrivateTmp=false
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

10、启动命令

```bash
$ chown -R mongodb:mongodb /jrtz/mongo/mongodb/log/mongodb.log
$ systemctl daemon-reload && systemctl enable mongodb && systemctl restart mongodb
```

11、检查

```bash
$ tail -f -n 500 /jrtz/mongo/mongodb/log/mongodb.log
$ journalctl -f -n 500 -u mongodb.service 
$ systemctl status mongodb
```



### 远程连接mongodb
 官网下载robo 3t  [https://robomongo.org/download](https://links.jianshu.com/go?to=https%3A%2F%2Frobomongo.org%2Fdownload)
 安装完后配置。



![img](https:////upload-images.jianshu.io/upload_images/2103305-32da1842deecb61d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/657/format/webp)

点击creat

![img](https:////upload-images.jianshu.io/upload_images/2103305-64f76a9acb8f5e14.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/543/format/webp)

请原封不动填写



![img](https:////upload-images.jianshu.io/upload_images/2103305-5030658381e9e09f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/549/format/webp)

切换到ssh选项卡

![img](https:////upload-images.jianshu.io/upload_images/2103305-2db7e4e964fd5275.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/548/format/webp)

按图设置

点save保存

![img](https:////upload-images.jianshu.io/upload_images/2103305-a1d94946ee09cb29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/654/format/webp)

点连接

![img](https:////upload-images.jianshu.io/upload_images/2103305-edb4bcd27219bfca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/330/format/webp)

输入服务器的登录密码

![img](https:////upload-images.jianshu.io/upload_images/2103305-121f804dce8f1b48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/258/format/webp)

连接成功

10、如何关闭数据库
 查看pid

```bash
$ ps aux |grep mongodb
```



![img](https:////upload-images.jianshu.io/upload_images/2103305-35e105c0e31eca29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/864/format/webp)

pid

```bash
$ sudo kill 5314
```

即可关闭数据库

### 2018年7月30日补充：

授权登录
 在日常工作中我们不可能把数据库设置为免认证登录并暴露在公网下，所以我们需要为数据库添加用户名和密码，具体操作如下：（文章来自ChasenKaos，转发请注明。谢谢 原文：<https://www.jianshu.com/p/994bc7b19b26>）

1、修改前文提到的conf文件，命令如下：

```bash
$ cd /usr/mongodb
$ vim mongodb.conf
```

打开后如图：

![img](https:////upload-images.jianshu.io/upload_images/2103305-3127712b1e965dde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/803/format/webp)

image.png

我们把noauth那一行，前面加上#，注释掉。
 再在最后一行添加 auth = true
 完整代码如下：

```properties
port=27017 #端口
dbpath= /usr/mongodb/db #数据库存文件存放目录
logpath= /usr/mongodb/log/mongodb.log #日志文件存放路径
logappend=true #使用追加的方式写日志
fork=true #以守护进程的方式运行，创建服务器进程
maxConns=100 #最大同时连接数
#noauth = true #不启用验证
journal=true #每次写入会记录一条操作日志（通过journal可以重新构造出写入的数据）。
#即使宕机，启动时wiredtiger会先将数据恢复到最近一次的checkpoint点，然后重放后续的journal日志来恢复。
storageEngine=wiredTiger  #存储引擎有mmapv1、wiretiger、mongorocks
bind_ip = 0.0.0.0  #这样就可外部访问了，例如从win10中去连虚拟机中的MongoDB
auth = true #用户认证
```

保存退出。

2、关闭数据库，前文已经提到了方法，我这里只做操作，如图：



![img](https:////upload-images.jianshu.io/upload_images/2103305-af2dd4027d4e1041.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/873/format/webp)

image.png

3、启动数据库,请参照前文方法，如图：



![img](https:////upload-images.jianshu.io/upload_images/2103305-ca002bc6a994660d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/642/format/webp)

image.png

插曲：在添加用户名之前应该先执行./mongo命令先打开mongodb数据库
 来自网友@OldX_cea8

4、依次执行下列命令 添加用户名

```bash
#使用admin数据库
use admin
#给admin数据库添加管理员用户名和密码，用户名和密码请自行设置
db.createUser({user:"admin",pwd:"zaq1xsw2",roles:["root"]})
#验证是否成功，返回1则代表成功
db.auth("admin", "zaq1xsw2")
#切换到要设置的数据库,以test为例,此命令也可以创建
use test
#为test创建用户,用户名和密码请自行设置。
db.createUser({user: "test", pwd: "zaq1xsw2", roles: [{ role: "dbOwner", db: "test" }]})
#更改密码
db.changeUserPassword('quotation','quotationinvest0755'); 
#删除当前库的一个用户
db.dropUser(“XXXX”)
#删除当前库的所有用户
db.dropAllUser()
```

执行完后，ctrl + c结束shell，并通过关闭，打开进行重启数据库。

5、查看内存、连接数

```bash
use admin
db.auth("admin", "zaq1xsw2")
db.serverStatus().mem
db.serverStatus().connections
```





6、通过robo 3t连接。
 connection标签页

![img](https:////upload-images.jianshu.io/upload_images/2103305-3dab6623cc509678.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/537/format/webp)

connection标签页

authentication标签页

![img](https:////upload-images.jianshu.io/upload_images/2103305-62518bf6d0393f72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/544/format/webp)

authentication标签页

ssh标签页

![img](https:////upload-images.jianshu.io/upload_images/2103305-7a159b6f4f1a787b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/539/format/webp)

ssh标签页

点击save后，连接即可，如果出现报错，请核对自己输入的信息是否有误。

---------------------------------------------------------------------------------------------------------

作者：派大C

链接：https://www.jianshu.com/p/994bc7b19b26

来源：简书

简书著作权归作者所有，任何形式的转载都请联系作者获得授权并注明出处。