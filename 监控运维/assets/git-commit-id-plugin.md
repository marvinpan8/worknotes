# Maven插件之git-commit-id-plugin

SCM使用GIT而非SVN时，使用Maven发布，总是会出一些莫名其妙的问题，google查找原因,无意中看到了这个插件;

对于该插件,到目前为止,文档比较少,尤其是中文的文档;全部的信息都包含在项目说明文件中了;项目地址:

https://github.com/ktoso/maven-git-commit-id-plugin

作者说该插件类似于 buildnumber-maven-plugin 插件(关于该插件，可参考http://blog.csdn.net/u011453631/article/details/10394475)，并对其进行了扩展。buildnumber-maven-plugin 插件仅支持SVN和CVS，而该插件又支持git。

> git-commit-id-plugin is a plugin quite similar to https://fisheye.codehaus.org/browse/mojo/tags/buildnumber-maven-plugin-1.0-beta-4 fo example but as buildnumber only supports svn (which is very sad) and cvs (which is even more sad, and makes bunnies cry) I had to quickly develop an git version of such a plugin. 

对于英语不好的我来说,看英语很痛苦,为了不让自己在同一个地方痛苦两次,尝试在此记录下该插件的使用及其配置,方便自己,也方便其他英语不好的同仁们;如有歧义,请以原版文档为主.

```xml
<plugin>
	<groupId>pl.project13.maven</groupId>
	<artifactId>git-commit-id-plugin</artifactId>
	<version>2.1.5</version>
	<executions>
		<execution>
			<goals>
				<goal>revision</goal>
			</goals>
		</execution>
	</executions>
	<configuration>
		<!--日期格式;默认值:dd.MM.yyyy '@' HH:mm:ss z;-->
		<dateFormat>yyyyMMddHHmmss</dateFormat>
		<!--,构建过程中,是否打印详细信息;默认值:false;-->
		<verbose>true</verbose>
		<!-- ".git"文件路径;默认值:${project.basedir}/.git; -->
		<dotGitDirectory>${project.basedir}/.git</dotGitDirectory>
		<!--若项目打包类型为pom,是否取消构建;默认值:true;-->
		<skipPoms>false</skipPoms>
		<!--是否生成"git.properties"文件;默认值:false;-->
		<generateGitPropertiesFile>true</generateGitPropertiesFile>
		<!--指定"git.properties"文件的存放路径(相对于${project.basedir}的一个路径);-->
		<generateGitPropertiesFilename>git.properties</generateGitPropertiesFilename>
		<!--".git"文件夹未找到时,构建是否失败;若设置true,则构建失败;若设置false,则跳过执行该目标;默认值:true;-->
		<failOnNoGitDirectory>true</failOnNoGitDirectory>

		<!--git描述配置,可选;由JGit提供实现;-->
		<gitDescribe>
			<!--是否生成描述属性-->
			<skip>false</skip>
			<!--提交操作未发现tag时,仅打印提交操作ID,-->
			<always>false</always>
			<!--提交操作ID显式字符长度,最大值为:40;默认值:7;
				0代表特殊意义;后面有解释; 
			-->
			<abbrev>7</abbrev>
			<!--构建触发时,代码有修改时(即"dirty state"),添加指定后缀;默认值:"";-->
			<dirty>-dirty</dirty>
			<!--always print using the "tag-commits_from_tag-g_commit_id-maybe_dirty" format, even if "on" a tag.
				The distance will always be 0 if you're "on" the tag.
			-->
			<forceLongFormat>false</forceLongFormat>
		</gitDescribe>
	</configuration>
</plugin>
```
以上代码给出了插件的使用及属性使用,中文内容则是根据自己的理解翻译; 以下是从项目文档中翻译过来的属性解释:
configuration options depth

- **dotGitDirectory**-(默认值:${project.basedir}/.git)".git"文件夹路径；**在多模块项目中,可以使用以下写法得到上一级目录中的".git"文件夹:${project.basedir}/../.git;**

- **prefix**-(默认值:git)公开属性的命名空间,保持默认,无须指定;
- **dateFormat**-(默认值:dd.MM.yyyy '@' HH:mm:ss z)是SimpleDateFormat类使用的格式化标准;用于格式化"git.build.time"和"git.commit.time";
- **verbose**-(默认值:false)如果设置为true,则会打印出获取的所有属性信息;
- **generateGitPropertiesFile**-(默认值:false)强制生成"git.properties"文件;
- **generateGitPropertiesFilename**-(默认值:src/main/resources/git.properties)指定生成的属性文件的路径,相对于${project.basedir}来说;
- **skipPoms**-(默认值:true)如果是pom类型项目,是否跳过执行;
- **failOnNoGitDirectory**-(默认值:true)".git"文件夹未找到时,构建是否失败;若设置true,则构建失败;若设置false,则跳过执行该目标;
- **gitDescribe**: Worth pointing out is, that git-commit-id tries to be 1-to-1 compatible with git's plain output, even though the describe functionality has been reimplemented manually using JGit (you don't have to have a git executable to use the plugin). So if you're familiar with git-describe, you probably can skip this section, as it just explains the same options that git provides.
  - **abbrev**-(默认值:7)打印的object id的长度;
  - **dirty**-(默认值:"")当有代码未提交时,执行此次操作,信息输出时添加后缀;"-"符号不会自动添加,建议设定该属性时添加"-"前缀,和commit ID区分开来;
  - **tags**-(默认值:false);
  - **long**-(默认值:false)如果当前提交是tag，git-describe默认只会输出tag名称；使用该属性可以强制格式化为指定格式。例如：tagname-0-gc0ffebabe，注意此处 -0-，如果不使用forceLongFormat模式,则输出为:tagname;
  - **always**-(默认值:true)如果不能发现tag,则此次提交会打印object id;
  - **skip**-(默认值:false)若没有在构建中使用"git-describe"信息，you can opt to be calculate it;

> 典型输出的例子:v2.1.0-1-gf5cd254, where -1- means the number of commits away from the mentioned tag，"-gf5cd254"部分表示此次提交操作的ID的前7位字符(f5cd254)，请注意,包含前缀"g"是为了说明它是一个commit id，它不是object id的一部分，这是git的一个默认行为;abbrev=0是一个特殊情况.其隐藏了从"tag"到"object id"的部分,具体请参考项目文档;

本文只是简单的介绍了插件的基本配置,更多深入的用法,请查看项目代码及使用文档;

---
原文链接：https://blog.csdn.net/wangjunjun2008/article/details/10526151