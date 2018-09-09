---
title: Spring Boot（二十二）：spring session实现集群-redis
date: 2018-09-09 16:18:09
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

Spring Boot中spring session支持方式有：

- JDBC
- MongoDB
- **Redis**
- Hazelcast
- HashMap

<!-- more -->

session集群的解决方案：

1. 扩展指定server

   利用Servlet容器提供的插件功能，自定义HttpSession的创建和管理策略，并通过配置的方式替换掉默认的策略。缺点：耦合Tomcat/Jetty等Servlet容器，不能随意更换容器。

2. 利用Filter

   利用HttpServletRequestWrapper，实现自己的 getSession()方法，接管创建和管理Session数据的工作。spring-session就是通过这样的思路实现的。

pom文件引入：

~~~java
<!-- spring session -->
<dependency>
	<groupId>org.springframework.session</groupId>
	<artifactId>spring-session</artifactId>
</dependency>
<!-- redis -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-redis</artifactId>
</dependency>	
~~~

配置：

~~~java
# spring session使用存储类型
spring.session.store-type=redis
# spring session刷新模式：默认on-save
spring.session.redis.flush-mode=on-save
# session超时时间，单位秒
server.session.timeout=30
~~~

测试：

定义两个项目，更改配置文件`spring.session.store-type=redis`和`spring.session.store-type=none`比较两个文件的sessionID：

~~~java
@RequestMapping(value = "/index")
public String index(ModelMap map, HttpSession httpSession) {
    map.put("title", "第一个应用：sessionID=" + httpSession.getId());
    System.out.println("sessionID=" + httpSession.getId());
    return "index";
}
~~~

