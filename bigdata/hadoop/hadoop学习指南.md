# 大数据技术原理与应用 第二章 大数据处理架构Hadoop 学习指南

Ruan Rongcheng 2020年12月18日 http://dblab.xmu.edu.cn/blog/285/

---

本指南介绍Linux的选择方案，并详细指引读者根据自己选择的Linux系统安装Hadoop。请务必仔细阅读完[厦门大学林子雨编著的《大数据技术原理与应用》](http://dblab.xmu.edu.cn/post/bigdata/)第2章节，再结合本指南进行学习。



Hadoop是基于Java语言开发的，具有很好跨平台的特性。Hadoop的所要求系统环境适用于Windows，Linux，Mac系统，我们推荐选择使用Linux或Mac系统。Mac系统存在于苹果电脑上，由于Mac系统对硬件有定制化要求，没法在Windows上使用虚拟机和双系统来使用Mac系统,我们下面也会给出Mac系统安装Hadoop的相关教程。而Linux系统则可以在Windows上使用虚拟机或双系统安装使用。如果选择Linux，我们需要首先安装好Linux系统，然后在Linux系统的基础上，安装Hadoop。
本章需要用到的所有软件，可以到这些软件的官网下载，也可以直接[点击这里从百度云盘下载各个软件](https://pan.baidu.com/s/1mUR3M2U_lbdBzyV_p85eSA)（提取码：99bg）。

## 一、Linux的选择

在Linux系统各个发行版中，CentOS系统和Ubuntu系统在服务端和桌面端使用占比最高，网络上资料最是齐全，所以我们建议使用CentOS 6.4系统或Ubuntu LTS 14.04。

> **选择Ubuntu还是CentOS**
>
> 一般来说，如果要做服务器，我们选择CentOS或者Ubuntu Server；如果做桌面系统，我们选择Ubuntu Desktop。但是在学习Hadoop方面，虽然两个系统没有多大区别，但是我们强烈推荐新手读者使用Ubuntu操作系统。下面我们也会分别给出在CentOS和Ubuntu系统下安装Hadoop的教程。

下面我们给出两个系统的下载地址。

### （一）下载地址

整体的系统安装文件较大(>1G)，我们推荐使用支持断点下载的工具，比如迅雷，或者QQ旋风。点击下载工具链接，选择自己喜欢的下载工具

- [下载迅雷工具](http://121.192.176.203/down.sandai.net/thunderspeed/ThunderSpeed1.0.32.350.exe)
- [下载QQ旋风工具](http://121.192.176.204/dldir1.qq.com/invc/cyclone/QQDownload_Setup_48_773_400.exe)

安装完上面的下载工具后，记得关闭浏览器，再重新打开浏览器访问本网页，下载下面的系统安装文件。

如果您的电脑比较老或者内存小于2G，那么建议您选择32位系统版本的Linux。如果内存大于4G，那么建议选择64位系统版本的Linux

1. CentOS

   32位CentOs 6.4的下载地址:

   [普通下载](http://121.192.176.204/mirror.symnds.com/distributions/CentOS-vault/6.4/isos/i386/CentOS-6.4-i386-bin-DVD1.iso) | [迅雷下载](thunder://QUFodHRwOi8vMTIxLjE5Mi4xNzYuMjA0L21pcnJvci5zeW1uZHMuY29tL2Rpc3RyaWJ1dGlvbnMvQ2VudE9TLXZhdWx0LzYuNC9pc29zL2kzODYvQ2VudE9TLTYuNC1pMzg2LWJpbi1EVkQxLmlzb1pa) | [旋风下载](qqdl://aHR0cDovLzEyMS4xOTIuMTc2LjIwNC9taXJyb3Iuc3ltbmRzLmNvbS9kaXN0cmlidXRpb25zL0NlbnRPUy12YXVsdC82LjQvaXNvcy9pMzg2L0NlbnRPUy02LjQtaTM4Ni1iaW4tRFZEMS5pc28=)

   64位CentOs 6.4的下载地址:
   [普通下载](http://121.192.176.204/mirror.symnds.com/distributions/CentOS-vault/6.4/isos/x86_64/CentOS-6.4-x86_64-bin-DVD1.iso) | [迅雷下载](thunder://QUFodHRwOi8vMTIxLjE5Mi4xNzYuMjA0L21pcnJvci5zeW1uZHMuY29tL2Rpc3RyaWJ1dGlvbnMvQ2VudE9TLXZhdWx0LzYuNC9pc29zL3g4Nl82NC9DZW50T1MtNi40LXg4Nl82NC1iaW4tRFZEMS5pc29aWg==) | [旋风下载](qqdl://aHR0cDovLzEyMS4xOTIuMTc2LjIwNC9taXJyb3Iuc3ltbmRzLmNvbS9kaXN0cmlidXRpb25zL0NlbnRPUy12YXVsdC82LjQvaXNvcy94ODZfNjQvQ2VudE9TLTYuNC14ODZfNjQtYmluLURWRDEuaXNv)

2. Ubuntu(推荐使用该系统)
   32位Ubuntu LTS 14.04的下载地址:[点击下载](http://www.ubuntukylin.com/downloads/download.php?id=37)

   64位Ubuntu LTS 14.04的下载地址:[点击下载](http://www.ubuntukylin.com/downloads/download.php?id=38)

### （二）系统安装方式

> **选择虚拟机安装还是双系统安装**
>
> Linux系统的安装主要有两种方式：虚拟机安装和双系统安装,由于虚拟机安装和使用Linux的硬件配置比较高，我们建议电脑比较新或者配置内存4G以上的电脑可以选择虚拟机安装，电脑较旧或配置内存小于等于4G的电脑强烈建议选择双系统安装，否则，在配置较低的计算机上运行LInux虚拟机，系统运行速度会非常慢。鉴于目前教师和学生的计算机硬件配置一般不高，建议教师和学生在实践教学中也采用双系统安装。

1. 虚拟机安装

   [VirtualBox下载地址](https://download.virtualbox.org/virtualbox/6.1.4/VirtualBox-6.1.4-136177-Win.exe)

   请参考安装指南：

   - [在Windows中使用VirtualBox安装Ubuntu](http://dblab.xmu.edu.cn/blog/337-2/)
   - [在Windows中使用VirtualBox安装CentOS](http://dblab.xmu.edu.cn/blog/164/)

   如果您的Windows系统使用非官方的破解版本，那么有可能出现VirtualBox不能打开新任务的错误，请参考下面的解决指南：
   [解决VirtualBox不能打开新任务](http://dblab.xmu.edu.cn/blog/?p=225)

2. 双系统安装
   请参考安装指南：
   第一步：[制定U盘启动安装](http://jingyan.baidu.com/article/59703552e0a6e18fc007409f.html)
   第二步：[双系统安装](http://jingyan.baidu.com/article/dca1fa6fa3b905f1a44052bd.html)

   ### （三）熟悉 Linux系统的使用方法

   （1）上面完成了Linux系统的安装以后，如果读者是初次使用Linux系统，请熟悉一下Linux常用命令，参考链接：[Linux系统的常用命令](http://dblab.xmu.edu.cn/blog/1624-2/)
   （2）如果在上面步骤中，读者采用了虚拟机的方式安装了Linux系统，可以学习一下如何在Windows和Linux之间互相传输文件，参考链接：[在Windows系统中利用FTP软件向Ubuntu系统上传文件](http://dblab.xmu.edu.cn/blog/1608-2/)
   （3）在Linux系统中，经常需要解压缩文件，所以，读者需要学习文件的解压方法，参考链接：[Linux系统中下载安装文件和解压缩方法](http://dblab.xmu.edu.cn/blog/1606-2/)
   （4）在Linux系统中，经常需要编辑文件，所以，读者需要学习vim编辑器的使用方法，参考链接：[Linux系统中vim编辑器的安装和使用方法](http://dblab.xmu.edu.cn/blog/1607-2/)

## 二、Hadoop安装方式

Hadoop的安装方式有三种，分别是单机模式，伪分布式模式，分布式模式。

- 单机模式：单机模式：Hadoop 默认模式为非分布式模式（本地模式），无需进行其他配置即可运行。非分布式即单 Java 进程，方便进行调试。
- 伪分布式模式：Hadoop 可以在单节点上以伪分布式的方式运行，Hadoop 进程以分离的 Java 进程来运行，节点既作为 NameNode 也作为 DataNode，同时，读取的是 HDFS 中的文件。
- 分布式模式：使用多个节点构成集群环境来运行Hadoop。

### （一）、单机和伪分布式安装方式

1. 如果系统是Linux，请参照下面给出的教程进行安装：

   在Ubuntu系统上安装Hadoop请参考：
   （1）《大数据技术原理与应用（第2版）》教材请参考： [Hadoop安装教程-单机-伪分布式配置-Hadoop2.6.0(2.7.1)-Ubuntu14.04(16.04)](http://dblab.xmu.edu.cn/blog/install-hadoop/)
   （2）《大数据技术原理与应用（第3版）》教材（已经在2020年12月出版，[教材官网](http://dblab.xmu.edu.cn/post/bigdata3)）请参考：[Hadoop安装教程_单机/伪分布式配置_Hadoop3.1.3/Ubuntu18.04(16.04)](http://dblab.xmu.edu.cn/blog/2441-2/)

   在CentOS系统上安装Hadoop请参考：
   [Hadoop安装教程-伪分布式配置-CentOS6.4-Hadoop2.6.0](http://dblab.xmu.edu.cn/blog/install-hadoop-in-centos/)

   需要注意以下几点：
   - 系统用户名使用hadoop
   - 不要修改/etc/hosts 默认的localhost地址，如果已经修改请重新把127.0.0.1映射到localhost

2. 如果系统是Mac，请参照下面给出的链接进行安装：
   [Mac 安装Hadoop教程-单机-伪分布式配置](http://dblab.xmu.edu.cn/blog/820-2/)

### （二）、分布式安装方式

（1）在集群上分布式安装Hadoop，请参考：
[Hadoop集群安装配置教程_Hadoop2.6.0_Ubuntu/CentOS](http://dblab.xmu.edu.cn/blog/install-hadoop-cluster/)

（2）使用Docker搭建Hadoop分布式集群，请参考实验室博客文章《[使用Docker搭建Hadoop分布式集群](http://dblab.xmu.edu.cn/blog/1233/)》。

到此为止，Hadoop的安装指南已经结束，如果想学习第3章《Hadoop文件系统》，

请参考第3章的学习指南：[大数据技术原理与应用 第三章 学习指南](http://dblab.xmu.edu.cn/blog/290-2/)