# tomcat默认日志优化处理

http://www.eryajf.net/1663.html

正常情况下，tomcat默认的程序会输出一大堆的日志，而这些日志，对日常运维来说，帮助并不大，反而徒增不少的烦人文件。

这些都可以通过配置给优化掉。

## 1，针对logging.properties。

修改`conf/logging.properties`日志配置文件可以屏蔽掉部分的日志信息。

将level级别设置成WARNING就可以大量减少日志的输出，当然也可以设置成OFF，直接禁用掉。

```bash
可以直接用下边的内容替换掉原来的文件。如果也是tomcat8的话。
[root@node1 tomcat]$cat conf/logging.properties
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.  ############################################################
# Handler specific properties.
# Describes specific configuration info for Handlers.
############################################################ 

1catalina.org.apache.juli.AsyncFileHandler.level = OFF
1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
1catalina.org.apache.juli.AsyncFileHandler.prefix = catalina. 

java.util.logging.ConsoleHandler.level = OFF
java.util.logging.ConsoleHandler.formatter = org.apache.juli.OneLineFormatter 

############################################################
# Facility specific properties.
# Provides extra control for each logger.
############################################################ 

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].level = OFF
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].handlers = 2localhost.org.apache.juli.AsyncFileHandler 

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].level = OFF
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].handlers = 3manager.org.apache.juli.AsyncFileHandler 

org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].level = OFF
org.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/host-manager].handlers = 4host-manager.org.apache.juli.AsyncFileHandler 

# For example, set the org.apache.catalina.util.LifecycleBase logger to log
# each component that extends LifecycleBase changing state:
# org.apache.catalina.util.LifecycleBase.level = FINE 

# To see debug messages in TldLocationsCache, uncomment the following line:
# org.apache.jasper.compiler.TldLocationsCache.level = FINE
```

## 2，关闭localhost_access_log日志。

修改在tomcat的安装目录conf文件夹下server.xml里配置，将AccessLogValve注释掉。

```xml
<!-- Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs" 
                prefix="localhost_access_log" suffix=".txt"               
				pattern="%h %l %u %t "%r" %s %b" / -->
```

一般这段代码的位置就在配置的底部，注释掉。

## 3，管理catalina.out

这个文件是tomcat的启动日志，很多时候可以帮助我们运维，可以留下来。

如果需要关闭，则可以设置 `bin/catalina.sh` 文件：

搜索：

```bash
if [ -z "$CATALINA_OUT" ] ; then  
	CATALINA_OUT="$CATALINA_BASE"/logs/catalina.out
fi
```

可以更改后边的输出路径为`/dev/null`，让启动日志从此无影无踪。

但是一般**不建议**关闭这条日志输出，而时间长了之后，这个文件又会非常大，可以通过定时任务的策略进行定期清空处理：

crontab -e 使用VISUAL或者EDITOR环境变量所指的编辑器编辑当前的crontab文件。当结束编辑离开时，编辑后的文件将自动安装。 

-l 在标准输出上显示当前的crontab。 
-r 删除当前的crontab文件。

```bash
$ crontab -e 

#clean the log, every 4 hours 
0 */4 * * * echo a > /jrtz/tomcat/tomcat-8.5.41/logs/catalina.out
```