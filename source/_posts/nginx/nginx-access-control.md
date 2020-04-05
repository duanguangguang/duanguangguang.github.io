---
title: nginx访问控制
date: 2020-03-28 14:33:19
categories: nginx
tags:
  - nginx
---

### 介绍

Nginx的访问控制分为：

- 基于IP的访问控制

  `http_access_module`。

- 基于用户的信任登录

  `http_auth_basic_module`。

<!-- more -->

### 一、基于IP的访问控制

配置语法：

~~~java
Syntax:	allow address|CIDR|unix:|all;
Default: --
Context: http,server,location,limit_except
~~~

- address

  表示ip地址。

- CIDR

  表示网段。例：`192.168.1.0-24`。

- unix

  主要在linux，unix上面用到的socket方式的访问。

- all

  允许所有的。

与之对应的不允许访问的配置语法：

~~~java
Syntax:	deny address|CIDR|unix:|all;
Default: --
Context: http,server,location,limit_except
~~~

### 二、基于IP的访问控制演示

1. 查询自己本机的出口ip

   ~~~java
   https://www.ip138.com/ // 公网查看本机ip
   ipconfig/ifconfig // 内网环境查看本机ip
   ~~~

2. 新建admin.html

   ~~~java
   <html>
   <head>
   	<meta charset="utf-8">
   	<title>admin</title>
   </head>
   <body style="background-color:red;">
   <h1>Admin</h1>
   </body>
   </html>
   ~~~

3. 修改配置文件

   ~~~java
   [root@node1 conf.d]# mv default.conf access_mod.conf
   location ~ ^/admin.html {
           root   /opt/app/code;
           deny 192.168.x.xxx;
           allow all;
           index  index.html index.htm;
       }
   ~~~

4. 重载nginx服务

   ~~~java
   [root@node1 conf.d]# nginx -t -c /etc/nginx/nginx.conf 
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   [root@node1 conf.d]# nginx -s reload -c /etc/nginx/nginx.conf 
   ~~~

5. 访问测试

   ~~~java
   403 Forbidden
   nginx/1.16.1
   ~~~

6. 测试只允许信任的ip访问

   ~~~java
   location ~ ^/admin.html {
           root   /opt/app/code;
           allow 192.168.x.0/24;  // 配置ip段方式
           deny all;
           index  index.html index.htm;
       }
   ~~~

### 三、基于IP的访问控制局限性

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-access-control\基于ip访问控制局限性.png)

Nginx基于ip的访问控制的原理是基于客户端的ip，但是对于nginx来说，他不知道哪个是真正的客户端。如果我们的访问不是客户端与服务端直接连接，而是通过了一层代理（nginx，7层负载均衡，CDN），因为`http_access_module`是基于`remote_addr`来进行识别客户端的ip。如上图，IP1是客户端，IP3是服务端，而IP1通过IP2去访问IP3的时候，`remote_addr`就识别的是IP2。也就是说我们想对客户端IP1进行访问限制是没有起到作用，反而是对中间的代理IP2进行了限制。所以说他的准确率是不高的。

如何解决这个问题呢？

1. 方法一

   采用别的HTTP头信息控制访问，如：`http_x_forwarded_for`。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-access-control\http_x_forwarded_for.png)

   从上图可以看出：

   ```java
   http_x_forwarded_for=IP1,IP2
   
   http_x_forwarded_for=ClientIP,Proxy(1)IP,Proxy(1)IP,...
   ```

2. 方法二

   结合geo模块做。

3. 方法三

   通过HTTP自定义变量传递。

### 四、基于用户的信任登录

配置语法：

~~~java
Syntax:	auth_basic string|off;
Default: auth_basic off;
Context: http,server,location,limit_except
~~~

- string

  这个字符串即表示了开启，又会在前端显示出了字符串的信息，也可以作为前端的登录提示。

~~~java
Syntax:	auth_basic_user_file file;
Default: --;
Context: http,server,location,limit_except
~~~

- file

  文件配置路径，用来作为认证存储用户名和密码信息的文件。[密码文件格式](http://nginx.org/en/docs/http/ngx_http_auth_basic_module.html )：

  ~~~java
  # comment
  name1:password1
  name2:password2:comment
  name3:password3
  ~~~

  需要先安装htpasswd，可以直接安装httpd-tools这个包：

  ~~~java
  [root@node1 conf.d]# yum install httpd-tools -y
  ~~~

  首次需要先生成密码文件：

  ~~~java
  [root@node1 nginx]# htpasswd -c ./auth_conf duan
  New password: 
  Re-type new password: 
  Adding password for user duan
  [root@node1 nginx]# more ./auth_conf 
  duan:$apr1$yBkwo1kS$vPr2sM.EfrcNcuzqSHfz9/
  ~~~

### 五、基于用户的信任登录演示

1. 修改配置文件

   ~~~java
   location ~ ^/admin.html {
           root   /opt/app/code;
   	    auth_basic "Auth access test!input your passward!";
   	    auth_basic_user_file /etc/nginx/auth_conf;	
           index  index.html index.htm;
       }
   ~~~

2. 重载nginx服务

   ~~~java
   [root@node1 conf.d]# nginx -t -c /etc/nginx/nginx.conf 
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   [root@node1 conf.d]# nginx -s reload -c /etc/nginx/nginx.conf 
   ~~~

3. 访问测试

   ~~~java
   http://192.168.x.xxx/admin.html // 需要输入用户名密码
   ~~~

### 六、基于用户的信任登录局限性

1. 用户信息依赖文件方式
2. 操作管理机械，效率低下

解决方式：

1. Nginx结合LUA实现高效验证
2. Nginx和LDAP打通，利用`nginx-auth-ldap`模块