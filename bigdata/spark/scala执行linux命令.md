# scala执行linux命令

https://www.jianshu.com/p/ac67e189c6ce

scala中执行外部命令(scala.sys.process)

目前 scala.sys.process 已经封装的足够简单。参考：[http://itang.iteye.com/blog/1126777](https://link.jianshu.com?t=http://itang.iteye.com/blog/1126777)

```scala
scala> import scala.sys.process._
# 只需在结尾用!号，就表示执行外部命令
scala> val list = "ls -l" !
```

还可以重定向，甚至可以在java对象与命令之间：

```scala
 scala> new java.net.URL("http://www.iteye.com") #>
 new java.io.File("/tmp/iteye.html") !
```

注意，重定向必须用 new java.io.File("") 封装，否则会当作命令，比如

```scala
 scala> "ls" #> "/tmp/a" !
```

将会出错，必须

```scala
 scala> "ls" #> new java.io.File("/tmp/a") !
```

管道的用法：

```scala
 scala> val list = "ls -l" #|  "grep P" !
```

**不能在命令表达式中直接用管道， 比如 "ls | grep XXX" 这样不灵，必须用 #| 声明。**

更多参考：[https://github.com/harrah/xsbt/wiki/Process](https://link.jianshu.com?t=https://github.com/harrah/xsbt/wiki/Process)

//2012.6.15
 要把System.getProperties 里的内容重定向到一个文件如何实现？
 下面的方法不行，它会将第一个表达式的结果当作命令来执行

```
 scala>  System.getProperties.toString #> new java.io.File("/tmp/env") !
```

直接将文字重定向到一个文件，我现在还不知道怎么做。只能变通用写文件的啰嗦方式。
