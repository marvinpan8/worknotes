# 采坑记录

## cas5.3.2单点登录-服务端集成shiro权限认证(五)

### 第二种方式：自定义登录验证集成shiro

- 所有文件一字不差

- 密码是明码

  

## cas5.3.2单点登录-集成客户端(六)

### 测试

启动服务端和客户端，此时访问 http 不是https
<http://app1.cas.com:8081/>  

## cas5.3.2单点登录-编写自己的cas-starter(九)

- 类Application 替换 `import priv.wangsaichao.cas.client.configuration.EnableCasClient`

- spring 启动tomcat7:run 启动有个坑：

  - 删除此`servlet-api`

  ```bash
  <dependency>
  	<groupId>javax.servlet</groupId>
  	<artifactId>servlet-api</artifactId>
  	<version>2.5</version>
  </dependency>
  ```

  - `cas-client-core` 增加排除

    ```bash
    <exclusions>
    	<exclusion>
    		<groupId>javax.servlet</groupId>
    		<artifactId>servlet-api</artifactId>
    	</exclusion>
    </exclusions>
    ```

    

## cas5.3.2单点登录-动态添加services(十三)

- 增加服务

  https://server.cas.com:8443/cas/addClient/app2.cas.com/10000002

- 删除服务

  https://server.cas.com:8443/cas/deleteClient/app2.cas.com/10000002