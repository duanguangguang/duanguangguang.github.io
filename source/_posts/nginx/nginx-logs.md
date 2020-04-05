---
title: nginx日志
date: 2020-03-28 14:31:57
categories: nginx
tags:
  - nginx
---

### 介绍

Nginx日志类型主要包括：

- error.log
- access_log

error.log主要记录nginx处理http请求的错误的状态，以及nginx服务本身运行的错误的状态。会按照不同的级别记录到error.log里面。

access_log会记录nginx每一次http请求的访问状态，主要用于分析每一次http的请求，以及客户行为进行分析。

<!-- more -->

### 一、实现方式

主要依赖于log_format，将日志变量记录到日志文件中。配置语法：

~~~java
Syntax:log_format name [escape=default|json]string ...;
Default:log_format combined "...";
Context:http //表明只能配置在http这个模块下面
~~~

1. error.log

   ~~~java
   error_log  /var/log/nginx/error.log warn;
   ~~~

   定义了error_log的日志路径以及日志级别。日志示例：

   ~~~java
   2020/03/28 20:44:32 [error] 8178#8178: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.x.xxx, server: localhost, request: "GET /favicon.ico HTTP/1.1", host: "192.168.1.104", referrer: "http://192.168.1.104/"
   ~~~

2. access_log

   ~~~java
   log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                     '$status $body_bytes_sent "$http_referer" '
                     '"$http_user_agent" "$http_x_forwarded_for"';
   
   access_log  /var/log/nginx/access.log  main;
   ~~~

   定义了access_log日志路径以及日志格式，main是定义的格式名称。日志示例：

   ~~~java
   192.168.x.xxx - - [28/Mar/2020:20:44:32 +0800] "GET / HTTP/1.1" 200 612 "-" "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.100 Safari/537.36" "-"
   ~~~

### 二、Nginx变量

1. http请求变量

   - arg_PARAMETER
   - http_HEADER
   - send_http_HEADER

   示例：记录用户的User-Agent字段

   ~~~java
   [root@node1 nginx]# curl -v www.baidu.com >/dev/null
   * About to connect() to www.baidu.com port 80 (#0)
   *   Trying 14.215.177.38...
   * Connected to www.baidu.com (14.215.177.38) port 80 (#0)
   > GET / HTTP/1.1
   > User-Agent: curl/7.29.0
   > Host: www.baidu.com
   > Accept: */*
   > 
   < HTTP/1.1 200 OK
   < Accept-Ranges: bytes
   < Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
   < Connection: keep-alive
   < Content-Length: 2381
   < Content-Type: text/html
   < Date: Sat, 28 Mar 2020 14:53:19 GMT
   < Etag: "588604dc-94d"
   < Last-Modified: Mon, 23 Jan 2017 13:27:56 GMT
   < Pragma: no-cache
   < Server: bfe/1.0.8.18
   < Set-Cookie: BDORZ=27315; max-age=86400; domain=.baidu.com; path=/
   < 
   ~~~

   - 配置nginx配置文件：编辑nginx.conf添加User-Agent变量：

     ~~~java
     log_format  main  '$http_user_agent' '$remote_addr - $remote_user [$time_local] 				  "$request" '
                       '$status $body_bytes_sent "$http_referer" '
                       '"$http_user_agent" "$http_x_forwarded_for"';
     ~~~

   - 检查配置文件：-t表示检查，-c表示配置文件路径：

     ~~~java
     [root@node1 nginx]# nginx -t -c /etc/nginx/nginx.conf 
     nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
     nginx: configuration file /etc/nginx/nginx.conf test is successful
     ~~~

   - 重新加载配置文件

     ~~~java
     [root@node1 nginx]# nginx -s reload -c /etc/nginx/nginx.conf 
     ~~~

     如果没有启动nginx服务，则先启动nginx：

     ~~~java
     [root@node1 nginx]# nginx -c /etc/nginx/nginx.conf 
     ~~~

   - 查看nginx服务，确保nginx成功启动

     ~~~java
     [root@node1 nginx]# ps -aux | grep nginx
     ~~~

   - 构造日志，请求多次

     ~~~java
     [root@node1 nginx]# curl http://127.0.0.1
     ~~~

   - 查看access_log

     ~~~java
     [root@node1 nginx]# tail -n 200 /var/log/nginx/access.log
     curl/7.29.0127.0.0.1 - - [28/Mar/2020:23:07:00 +0800] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
     curl/7.29.0127.0.0.1 - - [28/Mar/2020:23:07:02 +0800] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
     curl/7.29.0127.0.0.1 - - [28/Mar/2020:23:07:03 +0800] "GET / HTTP/1.1" 200 612 "-" "curl/7.29.0" "-"
     ~~~

2. 内置变量

   [官网地址](http://nginx.org/en/docs/syslog.html )

3. 自定义变量

### 三、log_format默认配置

~~~java
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';
~~~

1. remote_addr

   表示客户端地址。

2. remote_user

   表示http客户端请求nginx认证的用户名。

3. time_local

   表示nginx时间。

4. request

   表示request头的请求行。

5. status

   表示response返回的状态。

6. body_bytes_sent

   表示服务端响应给客户端body信息大小。

7. http_referer

   在防盗链或者用户行为分析中经常用到，表示上一级页面是哪个。

8. http_user_agent

   表示http头信息user_agent。指明客户端是用什么来访问的。

9. http_x_forwarded_for

   会记录每一级用户通过http请求里面对于所携带的http信息。

