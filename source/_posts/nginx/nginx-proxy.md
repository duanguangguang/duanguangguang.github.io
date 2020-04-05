---
title: nginx作为代理服务
date: 2020-03-31 23:22:52
categories: nginx
tags:
  - nginx
---

### 介绍

客户端往往无法向服务端直接发起请求的时候，就需要一个代理，代理就实现了客户端和服务端之间的通信。客户端首先会去请求代理，代理服务再去请求服务端，服务端再通过代理返回给客户端。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-proxy\01代理.png)

<!-- more -->

nginx作为代理服务，就可以实现很多类型的代理。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-proxy\02代理.png)

### 一、代理分类

1. 正向代理

   客户端请求代理服务，代理服务再请求服务端。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-proxy\正向代理.png)

   使用场景例：一个公司所有的电脑无法上网，只有一台电脑可以上网的时候，我们往往会在浏览器里面配置一个代理的地址，通过这台代理服务器去上公网。

   另一种常见的场景就是翻墙了，通过国外的一台代理，搜索我们想看的网站。 

2. 反向代理

   客户端去请求一个网站的时候，比如淘宝网，你不知道他后台有多少服务器，其实请求的往往是一个代理，这个代理就会发给对应的服务器。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-proxy\反向代理.png)

3. 代理区别

   区别在于代理的对象不一样：

   - 正向代理代理的对象是客户端

     比如说我们想去访问google，我们只需要把代理设置为对应的代理服务器即可。

   - 反向代理代理的对象是服务端

     我们并不需要去关心服务端具体是那一台服务器，反向代理会帮我们去处理请求。

### 二、配置语法

~~~java
Syntax:	proxy_pass URL; 
Default: --
Context: location,if in location,limit_except
~~~

url支持：`//http://localhost:8000/uri/ https://192.168.x.x:8000/uri/ http://unix:/tmp/backend.socket:/uri/`。

### 三、正向代理

1. 新建zheng.html

   ~~~java
   <html>
   <head>
       <meta charset="utf-8">
       <title>duan</title>
   </head>
   <body >
       <h1>Duan<h1>
   </body>
   </html>
   ~~~

2. 增加default.conf配置

   ~~~java
   location / {
           if ( $http_x_forwarded_for !~* "^192\.168\.x\.xxx") {
               return 403;
           }
           root   /opt/app/code;
           index  index.html index.htm;
       }
   ~~~

3. 在配置的192.168.x.xxx服务器上配置正向代理

   ~~~java
   resolver 8.8.8.8; //DNS解析
   location / {
           proxy_pass http://$http_host$request_uri;
       }
   ~~~

   - http_host

     就是我们要访问的hostName，主机名。

   - request_uri

     zheng.html。请求的request_uri。

4. 在浏览器配置http代理

5. 访问测试

   ~~~java
   http://192.168.x.xxx/zheng.html
   ~~~

### 四、代理配置语法

[代理配置语法补充](http://nginx.org/en/docs/http/ngx_http_proxy_module.html )

1. 缓冲区

   ~~~java
   Syntax:	proxy_buffering on|off; 
   Default: proxy_buffering on
   Context: http,server,location
   ~~~

   我们在代理服务器nginx server往后端真实的服务器转发请求的时候，proxy_buffering on打开的时候会尽可能的将把一个请求的所有信息收集完再返回给客户端，减少了频繁的IO损耗。

   扩展：`proxy_buffer_size`，`proxy_buffers`，`proxy_busy_buffers_size`。

2. 跳转重定向

   ~~~java
   Syntax:	proxy_redirect default; proxy_redirect off;proxy_redirect redirect replacement;
   Default: proxy_redirect default
   Context: http,server,location
   ~~~

   当我们用nginx代理服务器去代理后端的服务返回的是一个301重定向地址的时候，会把我们请求重定向到另一个地址中，返回给客户端。所以这种场景下，当后端返回的301地址是前端所无法访问到的，或需要nginx把服务端返回给客户端的地址做重写的时候，就用到了该配置。

3. 头信息

   ~~~java
   Syntax:	proxy_set_header field value; 
   Default: proxy_set_header Host $proxy_host;
   		 proxy_set_header Connection close;
   Context: http,server,location
   ~~~

   当头信息不准确时，可以使用proxy_set_header增加一个头，把对应的信息用新的头方式携带到后端。

   扩展：`proxy_hide_header`，`proxy_set_body`。

4. 超时

   ~~~java
   Syntax:	proxy_connect_timeout time; 
   Default: proxy_connect_timeout 60s;
   Context: http,server,location
   ~~~

   指的是nginx作为一个代理到后端服务的超时，是一个连接超时。

   扩展：`proxy_read_timeout`，`proxy_send_timeout`。

### 五、代理配置规范

企业常用配置项：

~~~java
proxy_pass http://127.0.0.1:8080;
proxy_redirect default;

proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;

proxy_connect_timeout 30;
proxy_send_timeout 60;
proxy_read_timeout 60;

proxy_buffer_size 32k;
proxy_buffering on;
proxy_buffers 4 128k;
proxy_busy_buffers_size 256k;
proxy_max_temp_file_size 256k;
~~~





