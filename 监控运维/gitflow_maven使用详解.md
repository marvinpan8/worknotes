# gitflow＋maven使用详解

本篇博文适用于已理解gitflow流程，想使用gitflow工具更好管理整个gitlow流程的读者。
关于什么是gitflow可以移步这里:http://www.ituring.com.cn/article/56870

## 目的
根据gitflow流程，每次开发从dev拉出feature分支，开发完成后合并到dev分支。之后使用gitflow自动完成下述流程：每次准备上线时从dev合并到master前拉release分支，将release合并到master并打tag，同时如有需要对pom中的包版本进行自动升级，并且对pom中的依赖进行校验，确保不依赖snapshot版本的包。

## 准备工作
1. 安装maven：在这里不详述了，有需要的读者可自行百度。
2. 安装gitflow：mac和Linux用户安装brew后可直接使用brew install git-flow安装
   ，windows用户可参考这里：http://my.oschina.net/xsjayz/blog/263059

## 使用jgitflow管理gitflow流程
1. 在pom文件中增加jgitflow插件。强烈建议使用1.0-m3版本，后续版本对git的版本有较强要求。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>external.atlassian.jgitflow</groupId>
            <artifactId>jgitflow-maven-plugin</artifactId>
            <version>1.0-m3</version>
        </plugin>
    </plugins>
</build>
```

2. 当一个功能开发完成后，手动将 feature 分支合并到 dev 分支，准备上线。

3. 自动拉release分支并升级版本号。确保pom中没有snapshot版本的依赖，在工程目录下执行命令：

   ```bash
   mvn jgitflow:release-start -DreleaseVersion=上线后的正式版本号 -DdevelopmentVersion=下一次使用的版本号 -DpushReleases=true -DallowSnapshots=true
   ```

   如当前工程是 1.3.0-SNAPSHOT 版本，则上线后版本应为 1.3.0 ,对应的下一次开发版本号为 1.4.0-SNAPSHOT，此时命令为：

   ```bash
   mvn jgitflow:release-start -DreleaseVersion=1.3.0 -DdevelopmentVersion=1.4.0-SNAPSHOT -DpushReleases=true -DallowSnapshots=true 
   ```

   **执行完成后，将在本地自动生成一个 release分支**，如图。

   ![img](assets/20160728193040286)

4. 将relase分支合并到master并打tag。执行命令

   ```bash
   mvn jgitflow:release-finish -DnoReleaseBuild=true -DnoDeploy=true -DpushReleases=true
   ```

   执行完成后relese自动合并到master并且打tag，如图

   ![这里写图片描述](assets/20160728193406323)

pom中的版本号也相应改变，这是此时dev分支的版本号：

![这里写图片描述](assets/20160728193542013)

这是master的版本号：

![这里写图片描述](assets/20160728193623914)

5.在master 分支上打包上线。

---

原文链接：https://blog.csdn.net/u011479540/article/details/52058175

