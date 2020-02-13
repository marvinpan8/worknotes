# **自定义JSP标签自动完成对页面按钮做权限拦截处理**

### 前提

许多后台系统的页面是使用JSP页面来编写的，在后台系统权限管理逐渐完善的过程中，就会引申出这个需求来：系统已经支持了指定请求的权限控制，能否在页面加载时就对无权限处理的按钮或链接进行隐藏，每次点击后提示无权限操作，这种体验实在是不好。

### 方案

答案是肯定可以支持的，而且实现起来也很容易。自定义一套jsp页面的标签，校验当前用户是否有某个权限点的访问权限作为标签的后台处理逻辑就可以了。

### Show me the code

接下来给出代码实现及核心注释，个别方法需要自己实现，比如获取当前用户id、判断一个用户是否有权访问某个权限点，这些方法各个系统是不同的。

首先定义一个后台处理标签渲染逻辑的类，同时继承`RequestContextAwareTag`类。有的学习教程写的是继承`TagSupport`，本质上是一样的，`RequestContextAwareTag`继承了`TagSupport`，并增加了获取了`RequestContext`的类，方便获取当前request等内容。

```java
package com.xxx.permission.tag;

import com.google.common.base.Splitter;
import lombok.Getter;
import lombok.Setter;
import org.apache.commons.lang.StringUtils;
import org.springframework.web.servlet.tags.RequestContextAwareTag;

import java.util.List;

public class AclCheckTag extends RequestContextAwareTag {

    /**
     * 权限标识码
     */
    @Getter
    @Setter
    private String code;

    public AclCheckTag(){
        super();
    }

    @Override
    protected int doStartTagInternal() throws Exception {
        if(needShowButton()) {
            return EVAL_BODY_INCLUDE; // Tag的关键字，代表将标签里的内容输出到存在的输出流中
        }
        return SKIP_BODY; // Tag的关键字，代表跳过开始和结束标签之间的代码
        /**
         * 其他关键字说明：
         * SKIP_PAGE： 忽略剩下的页面
         * EVAL_PAGE： 继续执行下面的页
         *
         * 对于控制按钮是否有权限展示，只需要 EVAL_BODY_INCLUDE 和 SKIP_BODY 即可
         */
    }

    private boolean needShowButton(){
        Integer currentUserId = getCurrentLoginUserId();
        if(currentUserId == null) { // 取不到登录用户，没权限，不展示按钮
            return false;
        }
        if(StringUtils.isEmpty(code)){ // 没有实际传入权限点，那么按照有权限来处理，也可以根据实际需要不允许访问，
            return true;
        }
        List<String> codeList = Splitter.on(",").omitEmptyStrings().trimResults().splitToList(code); // 支持逗号分隔多个权限点
        for(String aclCode : codeList){
            if(hasAcl(aclCode, currentUserId)){ // 这里假设拥有一个权限就代表有权限，也可以调整为需要拥有这里所有权限才代表有权限，可根据实际调整
                return true;
            }
        }
        return false;
    }

    private Integer getCurrentLoginUserId() {
        // TODO: 从上下文中取出当前登录的用户ID,
        return 1;
    }

    private boolean hasAcl(String aclCode, Integer userId) {
        // TODO：取出spring上下文，判断userId对应的用户是否可访问aclCode对应的权限点
        return true;
    }

    @Override
    public void release() {
        super.release();
        code = null;
    }

}
```

之后需要定义JSP页面使用的标签 acl-taglib.tld，习惯放在 /WEB-INF/tld 路径下。这个核心是指出标签逻辑处理的类及参数说明

```xml
<?xml version="1.0" encoding="UTF-8"?>

<taglib xmlns="http://java.sun.com/xml/ns/j2ee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee http://java.sun.com/xml/ns/j2ee/web-jsptaglibrary_2_0.xsd"
        version="2.0">
    <description>权限校验</description>
    <tlib-version>1.0</tlib-version>
    <short-name>acl</short-name>
    <uri>http://xxx.com</uri>
    <tag>
        <description>判断某个用户是否具有某个权限的按钮或链接</description>
        <name>checkPermission</name>
        <tag-class>com.xxx.tag.AclCheckTag</tag-class>
        <body-content>JSP</body-content>
        <attribute>
            <description>对应权限点的标识</description>
            <name>code</name>
            <required>true</required>
            <rtexprvalue>true</rtexprvalue>
            <type>java.lang.String</type>
        </attribute>
    </tag>
</taglib>
```

接下来在项目的web.xml里进行taglib的配置，注意 taglib-location 要和实际中放置的位置一致

```xml
<jsp-config> 
    <taglib>
        <taglib-uri>/acl</taglib-uri>
        <taglib-location>/WEB-INF/tld/acl-taglib.tld</taglib-location>
    </taglib>
</jsp-config>
```

## 使用

这些写好之后，接下来就可以在JSP页面使用了。

首先，使用标签时，JSP页面会要求你引入对应的taglib，这里需要注意uri的对应

```xml
<%@taglib prefix="acl" uri="/acl" %>
```

接下来就是在给指定的按钮配置标签和权限点啦，举个例子（这里假设我们举例子的按钮对应的权限点标识是 ACL0001）：

```xml
<acl:checkPermission code="ACL0001">
    <input type="submit"  name="submit" > 更新 </input>
</acl:checkPermission>
```

当然了，这里除了可以放置按钮外，任何HTML的片段都可以放到标签里，这里的后台逻辑是只关心当前用户是否有权访问配置的权限点，有权访问时才会输出标签里的HTML片段。使用时注意别把HTML片段对应的权限点标识搞错就可以了

## 延伸

如今，越来越多的系统采用前后端分离的架构，这时该如何在页面渲染时不展示无权操作的按钮和链接呢，这里给个简单的方案：

- 首先后台提供一个根据权限点标识（要支持多个权限点标识一起校验）检查当前用户是否有权访问的接口，
- 前台对于需要做权限校验的按钮先设置为不展示，然后在页面加载时发送请求到后台判断相关的权限标识是否可访问，前台拿到结果后对于有权访问的按钮和连接进行展示，无权访问的直接移除对应的dom元素。这样一来，效果就和权限标签做的事情是一样的了，只是多了个后台请求，这个请求即使被使用的人发现修改也没什么影响，因为正常每个请求还都会权限系统拦截住。

（完）



链接：https://www.imooc.com/article/20763