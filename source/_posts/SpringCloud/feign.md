---
title: Spring Cloud（五）：声明式Feign调用
date: 2018-10-04 18:23:16
tags:
 - SpringCloud
categories: 
 - SpringCloud
---

### 介绍

Feign是一个声明式的web服务客户端，它使得写web服务变得更简单。使用Feign,只需要创建一个接口并注解。
它具有可插拔的注解特性，包括Feign 注解和JAX-RS注解。
总结： 

1. Feign 采用的是基于接口的注解
2. Feign 整合了ribbon，具有负载均衡的能力

3. Feign 整合了Hystrix，具有熔断的能力

<!-- more -->

### feign调用

![](feign\feign01.png)

在调用的时候只需要把DeptClientService注册为bean，像调用本地方法一样。

### 服务降级

![](feign\feign02.png)

服务降级的逻辑的实现只需要为Feign客户端的定义接口编写一个具体实现类或者实现FallbackFactory，加上泛型，其中每个重写的方法的实现逻辑都可以用来定义相应的服务降级逻辑

### feign禁用hystrix

1. 禁用全局

   配置：feign.hystrix.enabled = false，可以解决Feign调用超时的问题

2. 禁用单个

   加个配置代码Feign.Builder，默认是HystrixFeifn.Builder

   ![](feign\feign03.png)

   ![](feign\feign04.png)

   ![](feign\feign05.png)

   ![](feign\feign06.png)

### 参考资料

1. [JAX-RS与JAX-WS]( https://www.zhihu.com/question/30095171/answer/46852591)