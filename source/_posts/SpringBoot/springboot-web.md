---
title: Spring Boot（七）：web开发之模板引擎
date: 2018-08-31 16:48:16
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

目前开发大都前后端分离，这里学习模板引擎只做了解。模板引擎的选择可以有：FreeMarker、Thymeleaf、Groovy、Mustache等。增加的css、js、等文件的目录结构：

![](springboot-web\fm01.png)

<!-- more -->

### FreeMarker

1. pom引用

   ~~~java
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-freemarker</artifactId>
   </dependency>
   ~~~

2. FreeMarkerController

   ~~~java
   @Controller
   @RequestMapping(value = "/web")
   public class FreeMarkerController {
       @RequestMapping(value = "index")
       public String index(ModelMap map) {//ModelMap选择可以有很多种，不做了解
           map.put("title", "freemarker hello world");
           return "index"; // 开头不要加上/，linux下面会出错
       }
   }
   ~~~

3. index.ftl

   ~~~html
   <!DOCTYPE html>
   <html>
       <head lang="en">
           <title>Spring Boot Demo - FreeMarker</title>
           <link href="/css/index.css" rel="stylesheet" />
       </head>
       <body>
           <center>
               <img src="/images/lele.jpg" />
               <h1 id="title">${title}</h1>
           </center>
   
          <script type="text/javascript" src="/webjars/jquery/2.1.4/jquery.min.js">		</script>
           <script>
               $(function(){
                   $('#title').click(function(){
                   alert('点击了');
                   });
               })
           </script>
       </body>
   </html>
   ~~~

### thymeleaf

1. pom引用

   ~~~java
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-thymeleaf</artifactId>
   </dependency>
   ~~~

2. index.html

   ~~~html
   <!DOCTYPE html>
   <html>
       <head lang="en">
           <title>Spring Boot Demo - thymeleaf</title>
           <link href="/css/index.css" rel="stylesheet" />	
       </head>
       <body>
           <center>
               <img src="/images/lele.jpg" />
               <h1 id="title" th:text="${title}"></h1>
           </center>
       </body>
   </html>
   ~~~

### jsp

springboot不建议使用：

-  jsp只能打包为：war格式，不支持jar格式，只能在标准的容器里面跑（tomcat，jetty都可以） 
-  内嵌的Jetty目前不支持JSPs
-  Undertow不支持jsps
-  jsp自定义错误页面不能覆盖spring boot 默认的错误页面

1. pom引用

   ~~~java
   <dependency>
   	<groupId>org.apache.tomcat.embed</groupId>
   	<artifactId>tomcat-embed-jasper</artifactId>
   	<scope>provided</scope>
   </dependency>
   <dependency>
   	<groupId>javax.servlet</groupId>
   	<artifactId>jstl</artifactId>
   </dependency>
   ~~~

2. application.properties

   ~~~java
   spring.mvc.view.prefix: /WEB-INF/templates/
   spring.mvc.view.suffix: .jsp
   ~~~

3. index.jsp

   ~~~java
   <%@ taglib prefix="spring" uri="http://www.springframework.org/tags"%>
   <%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
   <!DOCTYPE html>
   <html>
       <head lang="en">
           <title>Spring Boot Demo - FreeMarker</title>
           <link href="/static/css/index.css" rel="stylesheet" />
       </head>
       <body>
           <img src="/static/images/lele.jpg" alt="logo"/>
           <h1 id="title">${title}</h1>
           <c:url value="http://www.baidu.com" var="url"/>
         <spring:url value="http://www.baidu.com" htmlEscape="true" var="springUrl"/>
           Spring URL: ${springUrl}
           <br>
           JSTL URL: ${url}
       </body>
   </html>
   
   ~~~

4. ServletInitializer.java

   ~~~java
   import org.springframework.boot.builder.SpringApplicationBuilder;
   import org.springframework.boot.web.support.SpringBootServletInitializer;
   /**
    * 在Servlet容器中部署WAR的时候，不能依赖于Application的main函数而是要以类似于web.xml文件配置的方式来启动Spring应用上下文<br/>
    * 所以此时需要声明这样一个类或者将应用的主类改为继承SpringBootServletInitializer也可以
    */
   public class ServletInitializer extends SpringBootServletInitializer {
   	@Override
   	protected SpringApplicationBuilder configure(SpringApplicationBuilder 																application) {
   		return application.sources(SpringBootDemo81Application.class);
   	}
   }
   ~~~

### 错误处理

增加错误处理页面

![](springboot-web\error01.png)

1. ErrorController

   BaseErrorController

   ~~~java
   @Controller
   @RequestMapping(value = "error")
   public class BaseErrorController implements ErrorController {
       private static final Logger logger = LoggerFactory.getLogger(BaseErrorController.class);
       @Override
       public String getErrorPath() {
           logger.info("出错啦！进入自定义错误控制器");
           return "error/error";
       }
       @RequestMapping
       public String error() {
           return getErrorPath();
       }
   }
   ~~~

   error.ftl

   ~~~html
   <!DOCTYPE html>
   <html>
       <head lang="en">
           <title>Spring Boot Demo - error</title>
       </head>
       <body>
       	<h1>error-系统出错，请联系后台管理员</h1>
       </body>
   </html>
   ~~~

2. 自定义错误页面

   - html静态页面：在resources/public/error/ 下定义，如添加404页面： resources/public/error/404.html页面，中文注意页面编码
   - 模板引擎页面：在templates/error/下定义，如添加5xx页面： templates/error/5xx.ftl

   注：templates/error/ 这个的优先级比较 resources/public/error/高

   WebController

   ~~~java
   @Controller
   @RequestMapping(value = "/web")
   public class WebController {
       private org.slf4j.Logger logger = org.slf4j.LoggerFactory.getLogger(WebController.class);
       @RequestMapping(value = "error")
       public String error(ModelMap map) {
           throw new RuntimeException("测试异常");
       }
   }
   ~~~

3. @ControllerAdvice

   WebController同上

   BizException

   ~~~java
   @ControllerAdvice
   public class BizException {
       private static final Logger logger = LoggerFactory.getLogger(BizException.class);
       /**
        * 统一异常处理
        */
       @ExceptionHandler({ RuntimeException.class })
       @ResponseStatus(HttpStatus.OK)
       public ModelAndView processException(RuntimeException exception) {
           logger.info("自定义异常处理-RuntimeException");
           ModelAndView m = new ModelAndView();
           m.addObject("demoException", exception.getMessage());
           m.setViewName("error/500");
           return m;
       }
   
       /**
        * 统一异常处理
        */
       @ExceptionHandler({ Exception.class })
       @ResponseStatus(HttpStatus.OK)
       public ModelAndView processException(Exception exception) {
           logger.info("自定义异常处理-Exception");
           ModelAndView m = new ModelAndView();
           m.addObject("demoException", exception.getMessage());
           m.setViewName("error/500");
           return m;
       }
   }
   ~~~

   500.ftl

   ~~~html
   <!DOCTYPE html>
   <html>
   <head lang="en">
       <title>Spring Boot Demo - exception</title>
   </head>
   <body>
   <h1>500-系统错误</h1>
   <h1>${demoException}</h1>
   </body>
   </html>
   
   ~~~


