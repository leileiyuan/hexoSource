---
title: springMvc 响应返回406 Not Acceptable
date: 2016-12-17 10:46:30
tags:
 - spring
 - spring mvc
categories:
 - spring
 - spring mvc
---

请求地址，响应返回406 Not Acceptable

	http://sso.taotao.com/user/yuanleilei/1.html?r=0.950404558563605

web.xml中springmvc的配置

	<servlet>
		<servlet-name>taotao-web</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>classpath:spring/taotao-sso-servlet.xml</param-value>
		</init-param>
		<load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>taotao-web</servlet-name>
		<!-- 伪静态 伪静态有利于SEO（搜索引擎优化） -->
		<url-pattern>*.html</url-pattern>
	</servlet-mapping>

请求凡是以\*.html的结尾的都将被springmvc拦截，都认为是springmvc的合法请求，都会进入springmvc框架中进行处理。

后台日志中,请求已接受到，并由`UserController.check`方法处理，创建了事务`PROPAGATION_REQUIRED,ISOLATION_DEFAULT`，连接到数据库`jdbc:mysql://127.0.0.1:3306/taotao`，执行了查询`SELECT UPDATED,ID,USERNAME,EMAIL,PHONE,CREATED,PASSWORD FROM tb_user WHERE USERNAME = ? `，视图解析时发生异常`Resolving exception from handler......`

	 org.springframework.web.HttpMediaTypeNotAcceptableException: Could not find acceptable representation

```
2016-12-17 09:56:09,804 [http-bio-8083-exec-8] [org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver]-[DEBUG] Resolving exception from handler [public org.springframework.http.ResponseEntity<java.lang.Boolean> com.taotao.sso.controller.UserController.check(java.lang.String,java.lang.Integer)]: org.springframework.web.HttpMediaTypeNotAcceptableException: Could not find acceptable representation
2016-12-17 09:56:09,805 [http-bio-8083-exec-8] [org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver]-[DEBUG] Resolving exception from handler [public org.springframework.http.ResponseEntity<java.lang.Boolean> com.taotao.sso.controller.UserController.check(java.lang.String,java.lang.Integer)]: org.springframework.web.HttpMediaTypeNotAcceptableException: Could not find acceptable representation
2016-12-17 09:56:09,805 [http-bio-8083-exec-8] [org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver]-[DEBUG] Resolving exception from handler [public org.springframework.http.ResponseEntity<java.lang.Boolean> com.taotao.sso.controller.UserController.check(java.lang.String,java.lang.Integer)]: org.springframework.web.HttpMediaTypeNotAcceptableException: Could not find acceptable representation
2016-12-17 09:56:09,805 [http-bio-8083-exec-8] [org.springframework.web.servlet.DispatcherServlet]-[DEBUG] Null ModelAndView returned to DispatcherServlet with name 'taotao-web': assuming HandlerAdapter completed request handling
2016-12-17 09:56:09,805 [http-bio-8083-exec-8] [org.springframework.web.servlet.DispatcherServlet]-[DEBUG] Successfully completed request
 
```

原因：响应时设置了`@RequestBody`，把对象转换成json返回时，缺少依赖的jar包。
加入依赖包：`jackson-core-asl-x.jar`，`jackson-mapper-asl-x.jar`问题解决.

###但是。问题还是存在
**springmvc有个规定，html的请求不以json返回。**

我们配置的web.xml中，是以html为映射的

**现在有两种解决方法：**
1。把web.xml的映射方式改为别的，比如.do，所有依赖这个映射的url都要以\*.do结尾
2。加一个映射，

	<servlet-mapping>
		<servlet-name>taotao-web</servlet-name>
		<!-- 伪静态 伪静态有利于SEO（搜索引擎优化） -->
		<url-pattern>*.html</url-pattern>
	</servlet-mapping>
	<servlet-mapping>
		<servlet-name>taotao-web</servlet-name>
		<url-pattern>/service/*</url-pattern>
	</servlet-mapping>

以/service开头的请求和以\*.html结尾的请求，都是指向`taotao-web`的sevlet的。
那么我们的请求地址可以是

	http://sso.taotao.com/service/user/yuanleilei/1?r=0.950404558563605

**需要返回json的请求 加上/service/..去掉.html。访问页面加上.html。请求风格也保持了统一。**
