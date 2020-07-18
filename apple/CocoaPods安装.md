# 最新的CocoaPods的使用教程(一)

# 前言**

前几天发布我的开源库[<最简单方便的iOS轮播开源库：JYCarousel>](https://link.jianshu.com?t=https://github.com/Delyer/JYCarousel)到CocoaPods的时候。对CocoaPods重新学习了一下，之前只是会简单的使用，并没有全面的了解。现在要对它做一个**学习记录**吧，现在我还是只会简单的使用_，教程只是我夸大的说法（别骂我）。

下面的操作都是经过亲自验证通过的，放心操作！Cocoapods这部分知识一共有三篇博客：

> **1.CocoaPods的日常使用**
>
> **2.创建CocoaPods的私有库**
>
> **3.创建CocoaPods的开源库**

那就跟我一起来学习一下CocoaPods吧

# **一. CocoaPods的介绍**

什么是**CocoaPods**？**CocoaPods**是一个负责管理iOS项目中第三方开源库的工具，CocoaPods的项目源码在[https://github.com/CocoaPods/Specs](https://link.jianshu.com?t=https://github.com/CocoaPods/Specs)上管理。

经过CocoaPods团队的不懈努力，2016年5月10号，CocoaPods终于在其[官方博客](https://link.jianshu.com?t=http://blog.cocoapods.org/CocoaPods-1.0/)上宣布正式发布CocoaPods 1.0。与此同时，公开了相应的Mac版App——[CocoaPods App 1.0](https://link.jianshu.com?t=http://blog.cocoapods.org/CocoaPods-App-1.0/) 。

CocoaPods App 1.0 的下载地址：[https://cocoapods.org/app](https://link.jianshu.com?t=https://cocoapods.org/app)

# **二. CocoaPods的安装**

### **1. 替换ruby源**

CocoaPods是基于ruby ecosystem的，需要ruby环境，使用ruby的gem命令。所以我们的系统要有ruby环境。然而Mac系统默认会安装好ruby环境。可在终端`ruby -v`查看ruby版本：

```
//查看ruby版本
ruby -v
//输出信息
ruby 2.0.0p648 (2015-12-16 revision 53162) [universal.x86_64-darwin15]
```

查看ruby源

```
gem sources -l
```

默认情况下，终端会显示下面：

```
*** CURRENT SOURCES ***
https://rubygems.org/
```

当然这个源在墙内是访问不到的,所以要更换到ruby-china的镜像

```
// 1.移除掉原有的源
gem sources --remove https://rubygems.org/
//2.淘宝的源已经不更新维护了，现在使用ruby-china的源哦
gem source -a https://gems.ruby-china.com
以下命令添加淘宝的源：（不建议继续使用）
gem sources -a https://ruby.taobao.org/
// 3.验证是否替换成功
gem sources -l
```

如果显示下面输出就说明正确：

```
*** CURRENT SOURCES ***
https://gems.ruby-china.com
```

### **2. 更新升级 Gem 版本**

Gem是管理Ruby库和程序的标准包，如果它的版本过低也可能导致安装失败，解决方案自然是升级Gem，执行下述命令即可:

```
// 更新升级gem，国内需要切换源
sudo gem update -n /usr/local/bin --system
```

查看gem版本

```
gem -v
2.7.7
```

### **3. 安装CocoaPods**

OS X 10.11 以前安装命令为：

```
sudo gem install cocoapod
```

Mac系统为**OS X EL Capitan**安装命令为：

```
//安装最新版本
sudo gem install -n /usr/local/bin cocoapods

//安装指定版本
sudo gem install -n /usr/local/bin cocoapods -v 1.0.0 

//安装最新的release beta版本
sudo gem install -n /usr/local/bin cocoapods --pre 
```

**如果你想卸载CocoaPods怎么办？**看下面：

```
//卸载CocoaPods
sudo gem uninstall cocoapods
```

### **4. 更新Podspec索引文件**

如果按照上面3个步骤没问题，用命令`pod --version`查看是否安装成功，如果成功会显示pod的版本。

**pod setup作用：**将所有第三方的Podspec索引文件更新到本地的`~/.cocoapods/repos`目录下

pod安装成功之后一个首先的操作就是执行命令（不是必须的）：

```
pod setup
```

所有的第三方开源库的Podspec文件都托管在[https://github.com/CocoaPods/Specs](https://link.jianshu.com?t=https://github.com/CocoaPods/Specs)

我们需要把这个Podspec文件保存到本地，这样才能让我们使用命令`pod search 开源库`搜索一个开源库，怎样才能把github上的Podspec文件保存本地呢？那就是 **pod setup**

执行pod setup时，CocoaPods 会将第三方的podspec索引文件更新到本地的`~/.cocoapods/repos`目录下

- 如果没有执行过 **pod setup**，那**用户根目录下～**找不到`.cocoapods/repos`目录的，没有创建这个目录。
- 如果执行 **pod setup**，并且命令没有执行成功，那么会创建`~/.cocoapods/repos`目录，只不过目录是空的。
- 如果执行 **pod setup**，并且命令执行成功，说明把github上的Podsepc文件更新到本地，那么会创建`~/.cocoapods/repos`目录，并且`repos目录`里有一个`master目录`，这个master目录保存的就是github上所有第三方开源库的Podspec索引文件。

但是第一次执行pod setup时，这个github的Podspec索引文件比较大，有 300M 左右(以后会越来越大的)，所以第一次更新时非常慢.要耐心等待......可以进去目录`~/.cocoapods/repos`使用命令`du -sh *`来查看下载文件的大小了

**怎么才能快点呢？**网上好多给出都是更换索引库的镜像，gitcafe
 和oschina， gitcafe已经被coding收购了（2016年3月份左右收购）。这两个我亲测，现在都不行了（可能是我网速不好，基本上就是连接失败，有空网速好点的时候我在测试一下）。所以还是别更换 CocoaPods 索引库的镜像了。

# **三. CocoaPods的使用**

### **1. 新建 Podfile文件**

使用时需要在你的项目根目录下新建一个名为`Podfile`的文件（文件名一定为Podfile，不能更改），将依赖的库名字依次列在文件中即可.

```
//进入项目的根目录
cd ~/Desktop/Projects/JYCocoaPodsTest
//新建一个名为Podfile的文件
touch Podfile
```

### **2. 编辑 Podfile文件**

CococaPods升级到1.0.0版本之后，Podfile文件的格式也发生了很大改变
 我来带领大家写一个完整的`Podfile文件`  ,各个选项的解释在文件后面一起解释。

假设我们的项目名称为`JYCocoaPodsTest`, 为`JYCocoaPodsTest`新建了一个`UITarget`。打开JYCocoaPodsTest.xcodeproj结构如下：



![img](https:////upload-images.jianshu.io/upload_images/970779-5c16cc65ee4e94f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

JYCocoaPodsTest.xcodeproj结构.png

**1. 简单的Podfile：**

```
platform :ios, '7.0'
inhibit_all_warnings!

xcodeproj 'JYCocoaPodsTest'

workspace 'JYCocoaPodsTest'
use_frameworks!

target 'JYCocoaPodsTest' do
  pod 'AFNetworking',
  pod 'JYCarousel', '0.0.1'
end
```

**2. 稍微复杂的Podfile：**

```
source 'ssh://git@gitlab.9ijx.com:9830/iOS/Specs.git'
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, '7.0'
use_frameworks!
inhibit_all_warnings!

workspace 'JYCocoaPodsTest'

target 'JYCocoaPodsTest' do

pod 'AFNetworking'

pod 'JYCarousel', '0.0.1'

pod 'WCJCache', :git => "http://gitlab.9ijx.com/iOS/WCJCache.git"


target :JYCocoaPodsTestUITests do
    inherit! :search_paths
    pod 'YYText'

end

end
```

**3. Podfile的语法解释：**

> **1. platform :iOS, '7.0'**

- 指定了开源库应该被编译在哪个**平台**以及**平台的最低版本**。
- 若不指定平台版本，官方文档里写明各平台默认值为`iOS：4.3`,`OS X：10.6`,`tvOS：9.0`,`watchOS：2.0` 

> **2. inhibit_all_warnings!**

- 屏蔽cocoapods库里面的所有警告
- 这个特性也能在子target里面定义,如果你想屏蔽某pod里面的警告也是可以的:

```
pod 'JYCarousel', :inhibit_warnings => true
```

> **3. xcodeproj**,现在被**project**代替，这个变量就别使用了

- 允许你指定需要链接的工程

> **4. use_frameworks!**
>  使用frameworks动态库替换静态库链接
>  (1)swift项目cocoapods 默认 use_frameworks!
>  (2)OC项目cocoapods 默认 #use_frameworks!

> **5. workspace **

- 指定应该包含所有projects的Xcode workspace.
- 如果没有显示指定workspace并且在Podfile所在目录只有一个project，那么project的名称会被用作于workspace的名称

> **6. project**

- 默认情况下是没有指定的，当没有指定时，会使用Podfile目录下与target同名的工程：(我们只有一个工程JYCocoaPodsTest)
   \# JYCocoaPodsTest这个Target只有在JYCocoaPodsTest工程中才会链接
   target 'JYCocoaPodsTest' do
   project 'JYCocoaPodsTest'
   ...
   end

> **5. target 'xxxx' do**
>  ** end**

- 指定特定Target的依赖库
- 可以嵌套子Target的依赖库

> **6. inherit! :search_paths**

- 明确指定继承于父层的所有pod，默认就是继承的

> **7. source**

- 指定specs的位置,自定义添加自己的podspec。公司内部使用
   cocoapods 官方source是隐式的需要的，一旦你指定了其他source 你就需要也把官方的指定上
   例如：

```
source 'ssh://git@gitlab.9ijx.com:9830/iOS/Specs.git'
source 'https://github.com/CocoaPods/Specs.git'
```

- 当我们使用

  ```
  pod install
  ```

  或者

  ```
  pod setup
  ```

  时，会自动在

  ```
  ~/.cocoapods/repo
  ```

  目录下更新项目需要的podspec索引文件如下：

  

  ![img](../../../../Program Files/Typora)

  本地podspec索引文件.png

**4. 依赖库的基本写法：**

```
pod 'JYCarousel', //不显式指定依赖库版本，表示每次都获取最新版本
pod 'JYCarousel', '0.01'//只使用0.0.1版本
pod 'JYCarousel', '>0.0.1' //使用高于0.0.1的版本
pod 'JYCarousel', '>=0.0.1' //使用大于或等于0.0.1的版本
pod 'JYCarousel', '<0.0.2' //使用小于0.0.2的版本
pod 'JYCarousel', '<=0.0.2' //使用小于或等于0.0.2的版本
pod 'JYCarousel', '~>0.0.1' //使用大于等于0.0.1但小于0.1的版本，相当于>=0.0.1&&<0.1
pod 'JYCarousel', '~>0.1' //使用大于等于0.1但小于1.0的版本
pod 'JYCarousel', '~>0' //高于0的版本，写这个限制和什么都不写是一个效果，都表示使用最新版本
```

**5. 依赖库的自定义写法**
 下面都会用到podspec文件，所以要熟悉这个文件的构成才可以.
 我会在下一篇博客编辑这个文件

- **Using the files from a local path (使用本地文件)**

```
pod 'JYCarousel', :path => '/Users/Dely/Desktop/JYCarousel'
```

> 我新建一个工程PodTest来演示
>  1.在podfile写好依赖路径

```
platform :ios, '7.0'
inhibit_all_warnings!
workspace 'PodTest'
target 'PodTest' do
 pod 'JYCarousel', :path => '/Users/Dely/Desktop/JYCarousel'
end
```

> 2.在你的本地pod库添加xxx.podspec文件，（**一定要注意是根目录添加**）
>  
>
> 
>
> ![img](../../../../Program Files/Typora)
>
> JYCarousel目录结构.png
>
> ）
>
>  3.编辑xxx.podspec文件
>
>  主要下面红框点，修改成你的本地路径:
>
> 
>
> ![img](https:////upload-images.jianshu.io/upload_images/970779-98f116f55d88a369.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)
>
> podspec.png
>
>  4.进入项目根目录进行安装
>
> ```
> pod install
> ```
>
> 
>
> ![img](https:////upload-images.jianshu.io/upload_images/970779-1fd908c6cfeed40a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)
>
> PodTest.xcworkspace.png

-  **From a podspec in the root of a library repository (引用仓库根目录的podspec)**
   在仓库根目录下加入podsepc文件上传到git上就可以了（比较简单）

```
使用仓库中的master分支:
pod 'JYCarousel', :git => 'https://github.com/Delyer/JYCarousel.git'

使用仓库的其他分支:
pod 'JYCarousel', :git => 'https://github.com/Delyer/JYCarousel.git' :branch => 'release'

使用仓库的某个tag:
pod 'JYCarousel', :git => 'https://github.com/Delyer/JYCarousel.git', :tag => '0.0.1'

或者指定一个提交记录:
pod 'JYCarousel', :git => 'https://github.com/Delyer/JYCarousel.git', :commit => '5e473f1e0530bb3799f2f0d70554b292570bd8f0'
```

需要特别注意的是，虽然这样将会满足任何在Pod中的依赖项通过其他Pods
 但是 **podspec必须存在于仓库的根目录** 中，如果根目录中没有存在这个podspec文件，你将不得不使用下面提到的几种方式之一

- **From a podspec outside a spec repository, for a library without podspec（在一个不带podsepec的库里引用外部的spec）**

  如果一个podspec能够从外部的仓库源的获取，设想一下，也通过HTTP来获取podspec:

```
  pod 'JSONKit', :podspec => 'https://example.com/JSONKit.podspec'
```

- **pod spec**
   使用一个在给定podspec中声明的Pod的依赖项。如果如果没有参数被传递，那么在Podfile根部的第一个podspec会被使用。它将会被库所在的工程所使用
   注意:这个不会包含哪些来自于podspec的资源而仅仅是来自于CocoaPods基础架构

  例子:

```
podspec
podspec :name => 'QuickDialog'
podspec :path => '/Documents/PrettyKit/PrettyKit.podspec'
```

**上面的基本够我们日常使用。如果你还想了解更多。请到CocoaPods官方博客学习：https://guides.cocoapods.org**

### **3. 安装依赖开源库**

第二步中我们编辑好的Podfile文件保存好，进入项目根目录执行命令

```
//进入项目的根目录
cd ~/Desktop/Projects/JYCocoaPodsTest
//安装依赖库
pod install
```





![img](https:////upload-images.jianshu.io/upload_images/970779-b8f45688d7d3ea7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/456/format/webp)

pod install.png

```
pod install
```

命令执行成功后，会看到项目根目录下多出

```
xxx.xcworkspace
```

、

```
Podfile.lock文件
```

和

```
Pods目录
```

。再看看刚才执行完pod install命令打印出来的内容的最后一行：From now on use CocoaPodsDemo.xcworkspace.提示我们从现在起，我们需要使用JYCocoaPodsTest.xcworkspace文件来开发。



### **4. 第三方库更新**

跟`pod install`相似的一个命令就是`pod update`.
 如果未指定特定版本的话,`pod update`将所有第三方框架更新到最新版本。

### **5. pod文件和命令说明**

> **-----------------------------新增文件------------------------------**
>  **1. Podfile文件**
>  项目的第三方库的依赖以及项目的基本配置

> **2. Podfile.lock文件**
>  最后一次更新Pods时, 保存所有第三方框架的版本号

> **3. pods目录**
>  保存通过pod install或者pod update下载下来的第三方开源库的源代码

> **4. xxx.xcworkspace文件**
>  重新生成一个工作空间，打开这个工程文件来进行开发

> **-----------------------------常用指令------------------------------**
>  **1. pod setup**
>  将所有第三方的Podspec索引文件更新到本地的~/.cocoapods/repos目录下,更新本地仓库。

> **2. pod repo update**
>  执行 pod repo update更新本地仓库，本地仓库完成后，即可搜索到指定的第三方库，作用类似pod setup。不过这个命令经常不单独调用。比如执行`pod setup`、`pod search`、`pod install`、`pod update`会默认执行`pod repo update`

> **3. pod search xxx**
>  查找某一个开源库。查找开源库之前，默认会执行pod repo update指令

> **4. pod list**
>  列出所有可用的第三方库.现在已经2.4W+了.还在不断地增长

> **5. pod install**

- 会根据Podfile.lock文件中列举的版本号来安装第三方框架
- 如果一开始Podfile.lock文件不存在, 就会按照Podfile文件列举的版本号来安装第三方框架
- 安装开源库之前, 默认会执行pod repo update指令

> **6. pod update**

- 将所有第三方框架更新到最新版本, 并且创建一个新的Podfile.lock文件
- 安装开源库之前, 默认会执行pod repo update指令

> **7. pod install --no-repo-update**
>  **8. pod update --no-repo-update**
>  安装开源库之前, 不会执行pod repo update指令

### **四. CocoaPods相关的两个目录**

- 目录`~/.cocoapods/repos/`这个目录存储远端的podspec文件到本地。master是所有第三方的pod spec索引文件。其他的使我们自定义的podspec索引文件。



![img](https:////upload-images.jianshu.io/upload_images/970779-00ff85c687506dde.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/767/format/webp)

~/.cocoapods/repos/目录.png

- 目录

  ```
  ~/Library/Caches/CocoaPods/
  ```

  这个目录就是缓存文件的存储目录。

  

  ![img](https:////upload-images.jianshu.io/upload_images/970779-e6b0dc56de43882d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/769/format/webp)

  ~/Library/Caches/CocoaPods/目录.png

如果我们使用`pod search xxx` 提示没有找到,但是我们这个第三方确实存在。
 1.我们可以使用`pod setup`更新本地pod spec索引文件。然后`pod search xxx`
 2.按照1的方法如果还是`pod search xxx`找不到，那我们就把~/Library/Caches/CocoaPods/的缓存文件删除。然后`pod setup`。最后`pod search xxx`这样应该就可以了

CocoaPods的常见错误：[http://blog.csdn.net/wangyanchang21/article/details/51437934](https://link.jianshu.com?t=http://blog.csdn.net/wangyanchang21/article/details/51437934)

# **五. CocoaPods的说明**

1、第三方库会被编译成.a静态库或者.framwork的动态链接库供我们真正的工程使用。
 CocoaPods会将所有的第三方库以target的方式组成一个名为Pods的工程，该工程就放在刚才新生成的Pods目录下。整个第三方库工程会生成一个名称为libPods.a的静态库提供给我们自己的CocoaPodsTest工程使用。
 对于资源文件，CocoaPods提供了一个名为Pods-resources.sh的bash脚本，该脚本在每次项目编译的时候都会执行，将第三方库的各种资源文件复制到目标目录中。

2、我们的工程和第三方库所在的工程会由一个新生成的workspace管理
 为了方便我们直观的管理工程和第三方库，CocoaPodsTest工程和Pods工程会被以workspace的形式组织和管理，也就是我们刚才看到的JYCocoaPodsTest.xcworkspace文件。

3、原来的工程设置已经被更改了，这时候我们直接打开原来的工程文件去编译就会报错，只能使用新生成的workspace来进行项目管理。

4、CocoaPods通过一个名为Pods.xcconfig的文件来在编译时设置所有的依赖和参数。

# **六. 总结：**

CocoaPods主要还是侧重于使用，网速太慢我亲测的都有点烦躁了。当然理解原理过程更好，我现在还是有很多地方不是很理解。希望共同努力吧。还是多看看官方文档来学习一下不了解的。如果你喜欢就请点个喜欢吧_

---

# Q&A

以下所有命令均在终端中执行。

1. **报错信息：**

```
 -bash: /usr/local/bin/pod: /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/bin/ruby: bad interpreter: No such file or directory
```

**解决方案：**

1、更新 gem：**sudo gem update --system**

执行此步报错信息：

```
ERROR:  While executing gem … (Errno::EPERM)
​    Operation not permitted @ rb_sysopen - /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/bin/gem
```

**解决方案：**

**sudo gem update -n /usr/local/bin --system**

2、删除gem源：**gem sources —remove https://ruby.taobao.org/**

3、修改gem源：**gem sources -a https://gems.ruby-china.com**

4、查看gem源是否是最新的：**gem sources -l**

5、升级cocoapods：**sudo gem install -n /usr/local/bin cocoapods —pre**

会出现这个报错    

```
“ERROR:  While executing gem ... (URI::InvalidURIError)
 URI must be ascii only "?gems=\u{2014}pre"”
```

> ​    解决：使用 **sudo gem install -n /usr/local/bin cocoapods** 就不会有报错了。

6、查看升级后的cocoapods版本：**pod —version**

---

原文链接：https://www.jianshu.com/p/dfe970588f95/