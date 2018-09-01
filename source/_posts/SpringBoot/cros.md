---
title: Spring Boot（九）：CORS支持
date: 2018-08-31 21:41:48
tags:
 - SpringBoot
categories: 
 - SpringBoot
---

### 介绍

Web 开发经常会遇到跨域问题，解决方案有： jsonp， iframe，CORS 等。

 CORS 与 JSONP 相比 ：

1. JSONP 只能实现 GET 请求，而 CORS 支持所有类型的 HTTP 请求。
2.  使用 CORS，开发者可以使用普通的 XMLHttpRequest 发起请求和获得数据，比起 JSONP 有更好的 错误处理。 
3.  JSONP 主要被老的浏览器支持，它们往往不支持 CORS，而绝大多数现代浏览器都已经支持了 CORS 。

CORS 的实现可以通过全局配置或是全局设置。

<!-- more -->

### 全局配置

~~~java

~~~

### 全局设置

~~~java

~~~

### 测试方法

~~~java

~~~



