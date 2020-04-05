---
title: nginx模块
date: 2020-03-28 14:32:09
categories: nginx
tags:
  - nginx
---

### 介绍

Nginx模块分为：

- Nginx官方模块
- 第三方模块

本节主要介绍Nginx官方模块。主要就是`--with`。

<!-- more -->

### 一、_sub_status模块

| 编译选项                       | 作用              |
| ------------------------------ | ----------------- |
| --with-http_stub_status_module | Nginx的客户端状态 |

这个模块主要用于展示nginx当前处理连接的状态，用于监控nginx当前的连接信息。配置语法：

~~~java
Syntax:	stub_status;
Default: --
Context: server,location
~~~

示例：default.conf

~~~java
server {
    location /mystatus {
    	stub_status;
	}
}
~~~

1. 检查语法

   ~~~java
   [root@node1 conf.d]# nginx -tc /etc/nginx/nginx.conf 
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   ~~~

2. 重载服务

   ~~~java
   [root@node1 conf.d]# nginx -s reload -c /etc/nginx/nginx.conf
   ~~~

3. 访问测试，浏览器访问：

   ~~~java
   192.168.x.xxx/mystatus
   
   Active connections: 1  // nginx当前活跃的连接数
   server accepts handled requests
    1 1 1  // nginx处理的接收的握手的总次数 nginx所处理的连接数 总的请求数
   Reading: 0 Writing: 1 Waiting: 0 // 正在读的个数 正在往nginx写的个数 等待个数（主要是在nginx开启了keep alive长连接的情况下，客户端和服务端正在空闲的等待，建立连接的数量）
   ~~~

### 二、_random_index模块

| 编译选项                        | 作用                   |
| ------------------------------- | ---------------------- |
| --with-http_random_index_module | 目录中选择一个随机主页 |

这个模块主要用于在主目录中随机选择一个文件作为主页。配置语法：

~~~java
Syntax:	random_index on|off;
Default: random_index off;
Context: location
~~~

示例：

1. 新建3个html文件

   在/opt/app/code目录下新建red.html，black.html和blue.html。内容如下

   ~~~java
   <html>
   <head>
   	<meta charset="utf-8">
   	<title>duan1</title>
   </head>
   <body style="background-color:red;">
   </body>
   </html>
   ~~~

   将该目录配置在default.conf中。

2. 修改配置default.conf

   ~~~java
   server {
    	location / {
           root   /opt/app/code;
           random_index on;
           #index  index.html index.htm;
       }   
   }
   ~~~

3. 重载服务

   ~~~java
   [root@node1 conf.d]# nginx -tc /etc/nginx/nginx.conf 
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   [root@node1 conf.d]# systemctl reload nginx
   ~~~

4. 访问测试，多次访问观察页面颜色变化：

   ~~~java
   192.168.x.xxx
   ~~~

   nginx会随机选择一个html文件作为主页，但是不会选择一个隐藏文件，我们将blue.html文件隐藏测试：

   ~~~java
   [root@node1 conf.d]# cd /opt/app/code/
   [root@node1 code]# ls
   black.html  blue.html  red.html
   [root@node1 code]# mv blue.html .blue.html
   [root@node1 code]# ls
   black.html  red.html
   [root@node1 code]# ls -a
   .  ..  black.html  .blue.html  red.html
   ~~~

   再次访问则不会出现blue颜色的页面。

### 三、_sub_module模块

| 编译选项               | 作用         |
| ---------------------- | ------------ |
| --with-http_sub_module | http内容替换 |

这个模块用于nginx服务端在给客户端response http的内容的时候，用于对http的内容进行替换。应用场景：

nginx作为http server的时候，有多个虚拟主机，当我们需要对多个虚拟机返回的http内容进行替换时，一般依赖开发进行开发，周期是比较长的，但是用该模块可以将制定返回的内容替换成我们想要的内容，从而提高效率。配置语法：

~~~java
Syntax:	sub_filter string replacement; //string表示需要替换的内容，replacement表示替换后内容
Default: --
Context: http,server,location
~~~

还有一个就是nginx服务端用于完成和客户端（浏览器）进行每一次请求的时候校验服务端的内容是否有发生过变更。这个里面一般记录一串时间记录在http头信息里。

~~~java
Syntax:	sub_filter_last_modfiled on|off；
Default: sub_filter_last_modfiled off；
Context: http,server,location
~~~

还有一个就是匹配所有html代码里面的的第一个还是匹配所有指定的字符串，on表示匹配第一个。

~~~java
Syntax:	sub_filter_once on|off；
Default: sub_filter_once on；
Context: http,server,location
~~~

示例：sub_filter，sub_filter_once

1. 新建一个submodules.html文件

   ~~~java
   <html>
   <head>
   	<meta charset="utf-8">
   	<title>submodules</title>
   </head>
   <body>
   	<a>duan</a>
   	<a>hello</a>
   	<a>world</a>
   	<a>哈</a>
   	<a>哈</a>
   </body>
   </html>
   ~~~

   用于替换！。

2. 修改default.conf

   ~~~java
   location / {
           root   /opt/app/code;
           index  index.html index.htm;
           sub_filter '<a>哈' '<a>呵';
       }
   ~~~

3. 重载服务

   ~~~java
   [root@node1 conf.d]# nginx -tc /etc/nginx/nginx.conf 
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   [root@node1 conf.d]# nginx -s reload -c /etc/nginx/nginx.conf
   ~~~

4. 访问测试：

   ~~~java
   http://192.168.x.xxx/submodule.html // duan hello world 呵 呵
   ~~~



