---
title: Spring Boot（二十六）：基于HTTP的监控
date: 2018-09-09 17:03:19
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

利用Spring Boot的特性进行监控你的应用：

1. 通过HTTP（最简单方便）

2. 通过JMX

3. 通过远程shell

<!-- more -->

### 基于http的监控

pom文件引入：

~~~java
<!-- actuator -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- security -->
<dependency>
	 <groupId>org.springframework.boot</groupId>
	 <artifactId>spring-boot-starter-security</artifactId>
</dependency>
~~~

端点：

1. 端点暴露的方式取决于你采用的监控方式。如果使用HTTP监控，端点的ID映射到一个URL。例如，默认情况下，health端点将被映射到/health。

2. 端点会默认有敏感度，根据不同的敏感度是否需要提供用户密码认证

3. 如果没启用web安全，则敏感度高的会禁用

4. 可以通过配置文件进行配置敏感度

5. 默认情况下，除了shutdown外的所有端点都是启用的。

配置：

~~~java
#端点的配置
endpoints.sensitive=true
endpoints.shutdown.enabled=true

#保护端点
security.basic.enabled=true
security.user.name=dodd
security.user.password=****
management.security.roles=SUPERUSER

#自定义路径
security.basic.path=/manage
management.context-path=/manage
~~~

监控内容：

![](httpjk\httpjk01.png)

**度量**：[http://localhost:8080/manage/metrics](http://localhost:8080/manage/metrics)

**追踪**： [http://localhost:8080/manage/trace](http://localhost:8080/manage/trace)

