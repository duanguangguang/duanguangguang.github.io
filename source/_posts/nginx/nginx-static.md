---
title: nginx作为静态资源
date: 2020-03-31 23:22:42
categories: nginx
tags:
  - nginx
---

### 介绍

Nginx作为静态资源的HTTP server，他可以接收客户端 REQ:jpeg、htm、flv这种静态资源的请求，然后直接通过静态资源的存储，得到这些文件返回给客户端，这种方式是一个典型的比较高效的传输模式。这种场景常常会应用在静态资源的请求，处理，以及动静分离。那么什么是静态资源？

非服务器动态运行生成的文件。

| 类型         | 种类              |
| ------------ | ----------------- |
| 浏览器端渲染 | HTML、CSS、JS     |
| 图片         | JPEG、GIF、PNG    |
| 视频         | FLV、MPEG         |
| 文件         | TXT等任意下载文件 |

<!-- more -->

动态生成：我们需要进过服务端的解释器进行一些比较复杂的逻辑运算，对应的数据进行封装，再去返回给用户。

非服务器动态生成：客户端直接请求，服务端从文件系统直接就可以找到文件返回给用户。

### 一、CDN场景

CDN全称为内容分发网络，我们在请求静态资源的时候常常会用到。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\cdn.png)

对于CDN来说，我们要求的是传输延迟的最小化。

配置语法：sendfile

~~~java
Syntax:	sendfile on|off;
Default: sendfile off;
Context: http,server,location,if in location
~~~

对于nginx读取文件的方式，除了sendfile这种内核优化方式外，也用到了`--with-file-aio`异步文件读取方式。

配置语法：tcp_nopush

~~~java
Syntax:	tcp_nopush on|off;
Default: tcp_nopush off;
Context: http,server,location
~~~

作用：sendfile开启的情况下，提高网络包的传输效率。把多个包首体进行整合，一次性发送出去给用户端，这个对于报文的处理，提升了网络的传输的效率。

配置语法：tcp_nodelay

~~~java
Syntax:	tcp_nodelay on|off;
Default: tcp_nodelay off;
Context: http,server,location
~~~

作用：该网络包尽量不要等待，实时性的发送给用户，keepalive连接下，提高网络包的传输实时性。

配置语法：gzip压缩

```java
Syntax:	gzip on|off;
Default: gzip off;
Context: http,server,location,if in location
```

作用：压缩传输，需要对nginx进行静态资源的压缩，减小不必要的网络资源的消耗和传输的等待。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\gzip.png)

配置语法：gzip_comp_level压缩比

```java
Syntax:	gzip_comp_level level;
Default: gzip_comp_level 1;
Context: http,server,location
```

作用：level越大，文件压缩比率更大，传输的文件也就越小，但是压缩本身也需要消耗服务端的性能。

配置语法：gzip_comp_level http的版本

```java
Syntax:	gzip_http_version 1.0|1.1;
Default: gzip_http_version 1.1;
Context: http,server,location
```

作用：目前主流的http都是1.1的版本。

扩展Nginx压缩模块：

~~~java
http_gzip_static_module // 预读gzip功能
http_gunzip_module // 应用支持gunzip的压缩方式，为了解决很少一部分浏览器无法支持gzip压缩
~~~

场景演示：

新建code文件：

~~~java
[root@node1 log]# cd /opt/app/code/
[root@node1 code]# ls
admin.html  black.html  doc  download  images  red.html  submodule.html
[root@node1 code]# cd images/
[root@node1 images]# ls
wei.png
[root@node1 images]# cd ../doc
[root@node1 doc]# ls
access.txt
[root@node1 doc]# cd ../download/
[root@node1 download]# ls
test.img.gz
[root@node1 download]# 

~~~

- wei.png：验证图片压缩
- access.txt：验证文本压缩
- test.img.gz：验证download

配置文件：

~~~java
server {
    listen       80;
    server_name  192.168.x.xxx node1;
    
    sendfile on;
    #charset koi8-r;
    access_log  /var/log/nginx/log/static_access.log  main;

    
    location ~ .*\.(jpg|gif|png)$ {
        gzip on;
        gzip_http_version 1.1;
        gzip_comp_level 2;
        gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
        root  /opt/app/code/images;
    }

    location ~ .*\.(txt|xml)$ {
        gzip on;
        gzip_http_version 1.1;
        gzip_comp_level 1;
        gzip_types text/plain application/javascript application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
        root  /opt/app/code/doc;
    }

    location ~ ^/download {
        gzip_static on;
        tcp_nopush on;
        root /opt/app/code;
    }
~~~

在没有打开gzip的情况下和打开gzip情况下访问：

~~~java
http://192.168.x.xxx/wei.png
http://192.168.x.xxx/access.txt
http://192.168.x.xxx/test.img //当gzip_static off情况下回找不到文件报错，on会保存原文件
~~~

对比压缩前后的大小。

### 二、浏览器缓存原理

浏览器是有自己的缓存机制的，是基于HTTP协议的缓存机制（如：Expires，cache-control等）。实现浏览器的缓存就是依赖的HTTP头信息来跟服务端进行验证。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\浏览器缓存1.png)

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\浏览器缓存2.png)

校验过期机制：

| 校验是否过期            | Expires、Cache-Control(max-age) |
| ----------------------- | ------------------------------- |
| 协议中Etag头信息校验    | Etag                            |
| Last-Modified头信息校验 | Last-Modified                   |

1. 校验本地缓存是否过期（本地缓存文件里面的http头信息）

   - Expires：http1.0版本
   - Cache-Control：http1.1版本
   - max-age：周期，表示缓存文件在多久时间内表示过期

2. 如果本地缓存过期了，进行Etag验证和Last-Modified验证（相同，都是向服务端进行验证）

   向服务端先进行Etag验证，判断Etag是否过期，因为Last-Modified只能精确到秒，在1秒内的更新服务端是无法识别的。Etag是一串特殊的字符串，所以服务端也会保留一串特殊的字符串，来进行与Etag进行校验。

3. Last-Modified校验

   Last-Modified是一个具体的年与日时分秒，用来跟服务端文件和本地文件进行校验，如果本地文件在该时间周期有更新，那么服务端对应的文件的时间就会更新，出现客户端与服务端文件时间不一致，这样的话服务端就会返回最新的文件给客户端。

整体流程如下：

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\校验缓存过期.png)

配置语法：expires

~~~java
Syntax:	expires[modified]time;
        expires epoch|max|off;
Default: expires off
Context: http,server,location,if in location
~~~

expires的原理就是对客户端的response报文的头信息里面添加Cache-Control和Expires头。

场景演示：

新建code：test_expires.html

~~~java
<html>
<head>
	<meta charset="utf-8">
	<title>duan admin</title>
</head>
<body style="background-color:red;">
<h1>Hello Word!</h1>
</body>
</html>
~~~

增加配置：

~~~java
location ~ .*\.(htm|html)$ {
        #expires 24h;
        root /opt/app/code/html;
    }
~~~

访问测试：

~~~java
http://192.168.x.xxx/test_expires.html
~~~

第一次访问：

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\第一次访问.png)

第二次访问：

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\第二次访问.png)

- 第二次访问多了Cache-Control:max--age=0。这个是浏览器自己加进去的，max--age=0或者max--age<0表示每一次请求都要向服务端进行一次校验，看文件有没有更新，如果有更新则返回200，如果没有更新，则返回304

修改配置：打开过期时间

~~~java
location ~ .*\.(htm|html)$ {
        #expires 24h;
        root /opt/app/code/html;
    }
~~~

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\加过期时间访问.png)

- 发现在Response Headers里面多加了两个头
  - Cache-Control:max--age=86400（秒）
  - Expires: Thu, 02 Apr 2020 15:26:53 GMT
- 但是客户端浏览器遵循自己设置的Cache-Control:max--age=0，去请求服务端

### 三、跨站访问

理论上浏览器都会禁止网站进行一个页面进行跨站式访问。跨站式访问：浏览器访问一个服务端，一个页面里面当访问a时，同时会访问b（ajax），浏览器一般是会禁止的，主要是出于安全的考虑。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\跨站访问.png)

为什么浏览器禁止跨域访问？原因是不安全，容易出现CSRF攻击

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\禁止跨站访问.png)

一般一个用户去访问一个正规的网站A，A会返回给用户session，cookie等信息，存放在客户端，当客户一不小心点到了一些非法的网站B，这时网站B就可以给用户response一些带有恶意的请求，让用户去请求网站A，这样就形成了跨站访问。

打开跨站访问配置：

~~~java
Syntax:	add_header name value[always];
Default: --
Context: http,server,location,if in location
~~~

浏览器会判断头信息：`Access-Control-Allow-Origin`，看是否允许跨站访问。

跨站演示：

新建test_oringin.html：

~~~java
<html lang="en">  
<head>  
<meta charset="UTF-8" />  
<title>测试ajax和跨域访问</title>  
<script src="http://libs.baidu.com/jquery/2.1.4/jquery.min.js"></script>  
</head>  
<script type="text/javascript">  
$(document).ready(function(){  
    $.ajax({  
        type: "GET",  
        url: "http://node2/1.html",  
        success: function(data) {   
            alert("sucess!!!");  
        },  
        error: function() {  
            alert("fail!!!,请刷新再试!");  
        }  
    });  
});  
</script>  
<body>  
     <h1>测试跨域访问</h1>
</body>  
</html>  
~~~

新增配置：

~~~java
location ~ .*\.(htm|html)$ {
        #add_header Access-Control-Allow-Origin *; 
        #add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE,OPTIONS;
        #expires 24h;
        root  /opt/app/code/html;
    }
~~~

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\测试跨域访问.png)

### 四、防盗链

防盗链的目的是防止网站资源被盗用，防盗链设置的首要方式是区别哪些请求是非正常的用户请求。

基于http_refer防盗链配置模块。配置语法：

~~~java
Syntax:	valid_referers none|blocked|server_names|string...;
Default: --
Context: server,location
~~~

1. 新建test_refer.html

   ~~~java
   <html>
   <head>
       <meta charset="utf-8">
       <title>imooc1</title>
   </head>
   <body style="background-color:red;">
       <img src="http://192.168.x.xxx/wei.png"/>
   </body>
   </html>
   ~~~

   在不配置http_refer防盗链测试refer功能：

   ~~~java
   http://192.168.x.xxx/test_refer.html
   ~~~

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-static\test_refer.png)

2. 添加http_refer配置

   ~~~java
   location ~ .*\.(jpg|gif|png)$ {
       	#valid_referers none blocked 192.168.x.xxx ~/google\./; //支持正则方式
   	    valid_referers none blocked 192.168.x.xxx;
           if ($invalid_referer) {
               return 403;
           }
           root  /opt/app/code/images;
       }
   ~~~

   - valid_referers

     表示允许那些信息过来访问。

   - none

     表示允许没有带refer信息的请求过来。

   - blocked

     表示refer信息不是标准http://，也是不允许访问的。

   - ip

     表示只允许该ip过来访问。

   - $invalid_referer

     如果没有设置ip，$invalid_referer变量值就会为1，返回403。

3. 访问测试

   ~~~java
   // 不带refer信息
   [root@node1 conf.d]# curl -I http://192.168.x.xxx/wei.png
   HTTP/1.1 200 OK
   Server: nginx/1.16.1
   Date: Thu, 02 Apr 2020 15:21:43 GMT
   Content-Type: image/png
   Content-Length: 244044
   Last-Modified: Tue, 08 Aug 2017 09:17:25 GMT
   Connection: keep-alive
   ETag: "598981a5-3b94c"
   Accept-Ranges: bytes
   
   // 带refer信息
   [root@node1 conf.d]# curl -e "http://www.baidu.com" -I http://192.168.x.xxx/wei.png
   HTTP/1.1 403 Forbidden // 防盗链起作用
   Server: nginx/1.16.1
   Date: Thu, 02 Apr 2020 15:22:11 GMT
   Content-Type: text/html
   Content-Length: 153
   Connection: keep-alive
   
   // 改成配置的可访问的ip
   [root@node1 conf.d]# curl -e "http://192.168.x.xxx" -I http://192.168.x.xxx/wei.png
   HTTP/1.1 200 OK
   Server: nginx/1.16.1
   Date: Thu, 02 Apr 2020 15:23:17 GMT
   Content-Type: image/png
   Content-Length: 244044
   Last-Modified: Tue, 08 Aug 2017 09:17:25 GMT
   Connection: keep-alive
   ETag: "598981a5-3b94c"
   Accept-Ranges: bytes
   ~~~

   









