---
title: nginx与Lua开发
date: 2020-04-04 11:47:34
categories: nginx
tags:
  - nginx
---

### 介绍

Lua是一个简洁、轻量、可扩展的脚本语言。Nginx+Lua的优势：充分的结合Nginx的并发处理epoll优势和Lua的轻量实现简单的功能且高并发的场景。

<!-- more -->

### 一、Lua基础语法

1. 安装Lua解释器

   ~~~java
   [root@node1 /]# yum install lua
   
   [root@node1 /]# lua
   Lua 5.1.4  Copyright (C) 1994-2008 Lua.org, PUC-Rio
   > print("Hello,world")
   Hello,world
   > 
   ~~~

2. 脚本运行lua

   新建一个lua文件

   ~~~java
   #!/usr/bin/lua
   print("Hello,World!")
   ~~~

   赋权与运行：

   ~~~java
   [root@node1 lua]# chmod a+rx ./test.lua 
   [root@node1 lua]# ./test.lua 
   ~~~

3. lua注释

   ~~~java
   --行注释
   
   --[[
       块注释
   --]]
   ~~~

4. lua变量

   ~~~java
   a = 'alo\n123"' // 特殊符号，换行符
   a = '\97lo\10\04923"' //ASCLL
   a = [[alo123"]] //变量赋值
   ~~~

   布尔类型只有nil和false，数字0、空字符串都是true。

   lua中的变量如果没有特殊说明，全是全局变量。局部变量前加local。

5. while循环语句

   ~~~java
   sum= 0
   num = 1
   while num <= 100 do
       sum = sum + num
       num = num + 1
   end
   print("sum =",sum) 
   ~~~

   lua没有++或+=这样的操作。

6. for循环语句

   ~~~lua
   sum = 0
   for i = 1,100 do
       sum = sum + i
   end
   ~~~

7. 判断语句

   ~~~lua
   if age ==40 and sex == "Male" then
       print("大于40男子")
   elseif age > 60 and sex ~= "Female" then
       print("非女子而且大于60")
   else
       local age = io.read()
       print("Your age is"..age) // ..z字符串的拼接操作
   end
   ~~~

### 二、Nginx+Lua环境

默认情况下nginx是不支持lua的扩展模块。参考[Nginx编译安装Lua模块](https://www.imooc.com/article/19597 )

1. 下载LuaJIT：是一个解释器，比lua解释器高效

   ~~~java
   [root@node1 ~]# cd /opt/soft/backup/
   [root@node1 backup]# wget http://luajit.org/download/LuaJIT-2.0.2.tar.gz
   // 解压
   [root@node1 soft]# tar -zxvf LuaJIT-2.0.2.tar.gz 
   
   // 安装到指定目录
   [root@node1 LuaJIT-2.0.2]# make install PREFIX=/usr/local/LuaJIT
   
   // 导入环境变量
   [root@node1 LuaJIT-2.0.2]# vim /etc/profile
   export LUAJIT_LIB=/usr/local/LuaJIT/lib
   export LUAJIT_INC=/usr/local/LuaJIT/include/luajit-2.0
   ~~~

   

2. 下载nginx编译所需要的开发库以及对应的模块：ngx_devel+kit和lua-nginx-module

   ~~~java
   // 下载开发库和编译模块，分别解压
   [root@node1 backup]# wget https://github.com/simpl/ngx_devel_kit/archive/v0.3.0.tar.gz
   [root@node1 backup]# wget https://github.com/openresty/lua-nginx-module/archive/v0.10.9rc7.tar.gz
   ~~~

3. 重新编译Nginx

   ~~~java
   // 卸载nginx
   [root@node1 backup]# yum remove nginx
   // 查看nginx是否还存在
   [root@node1 backup]# which nginx
   // 重新安装nginx,下载nginx1.16.1解压到自定义安装目录
   [root@node1 nginx-1.16.1]# pwd
   /opt/soft/nginx/nginx-1.16.1
   // 编译
   ./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --add-module=/opt/soft/ngx_devel_kit/ngx_devel_kit-0.3.0 --add-module=/opt/soft/lua_nginx_module/lua-nginx-module-0.10.9rc7
   // 安装
   [root@node1 nginx-1.16.1]# make -j 4 && make install
   
   // 如果make提示：*** 没有规则可以创建“default”需要的目标“build”，一般是缺少相关依赖包，编译会提示not found，安装上相关的依赖包，重新编译安装
   yum install pcre-devel zlib zlib-devel openssl openssl-devel
   ~~~

4. 错误解决

   ~~~java
   // nginx检查
   [root@node1 nginx-1.16.1]# nginx -t
   nginx: error while loading shared libraries: libluajit-5.1.so.2: cannot open shared object file: No such file or directory
   
   // 这个问题需要将libluajit-5.1.so.2路径变量加入系统默认库
   [root@node1 etc]# cat /etc/ld.so.conf
   include ld.so.conf.d/*.conf
   [root@node1 etc]# ec
   [root@node1 etc]# echo “/usr/local/lib” >> /etc/ld.so.conf
   [root@node1 etc]# echo “/usr/localLuaJIT/lib” >> /etc/ld.so.conf
   
   // idconfig命令调用系统链接库
   [root@node1 sbin]# /sbin/ldconfig
   ~~~

5. 安装成功

   ~~~java
   [root@node1 nginx-1.16.1]# nginx -t
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   ~~~

### 三、Nginx调用Lua模块指令

Nginx设计成一种可插拔的模块化加载执行，共11个处理阶段。nginx调用lua指定：

| Lua模块指定         | 描述                                  |
| ------------------- | ------------------------------------- |
| set_by_lua          | 设置nginx变量，可以实现复杂的赋值逻辑 |
| set_by_lua_file     | 同上，lua可执行文件                   |
| access_by_lua       | 请求访问阶段处理，用于访问控制        |
| access_by_lua_file  | 同上                                  |
| content_by_lua      | 内容处理器，接收请求处理并输出响应    |
| content_by_lua_file | 同上                                  |

lua调用nginx指令，通过Nginx Lua API：

| Nginx Lua API        | 描述                                  |
| -------------------- | ------------------------------------- |
| ngx.var              | nginx变量                             |
| ngx.req.get.headers  | 获取请求头                            |
| ngx.rea.get_uri_args | 获取url请求参数                       |
| ngx.redirect         | 重定向                                |
| ngx.print            | 输出响应内容体                        |
| ngx.say              | 同ngx.print，但是最后会输出一个换行符 |
| ngx.header           | 输出响应头                            |
| ...                  | ...                                   |

### 四、灰度发布场景

按照一定的关系区别，分部分的代码进行上线，使代码的发布能平滑过渡上线。可以基于：

1. 用户的信息cookie等信息区别
2. 根据用户的ip地址

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-lua\灰度发布.png)

实战场景：代码1存放老代码，代码2存放新代码，通过ip过滤访问新老代码。

首先，安装memcached：

~~~java
[root@node1 /]# yum install memcached
~~~

1. 两个后台服务

   ~~~java
   [root@node1 soft]# ls
   LuaJIT  lua_nginx_module  nginx  ngx_devel_kit  pcre-8.44  tomcat8080  tomcat9090
   ~~~

   两个tomcat的/webapps/ROOT/都有一个java_test.jsp文件：

   ~~~java
   <%@ page language="java" import="java.util.*" pageEncoding="utf-8"%>  
   <HTML>
       <HEAD>
           <TITLE>JSP Test Page</TITLE>
       </HEAD>
       <BODY>
           <%
               Random rand = new Random();
   			//out.println("<h1>test server</h1>");
               out.println("<h1>Random number:</h1>");
               out.println(rand.nextInt(99)+100);
           %>
       </BODY>
   </HTML>
   ~~~

2. 启动tomcat

   ~~~java
   [root@node1 bin]# sh catalina.sh start;tail -f ../logs/catalina.out
   ~~~

   查看启动端口：

   ~~~java
   [root@node1 bin]# netstat -luntp
   Active Internet connections (only servers)
   Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
   tcp6       0      0 :::9090                 :::*                    LISTEN      30760/java                        
   tcp6       0      0 :::8080                 :::*                    LISTEN      31057/java 
   ~~~

3. 启动memcached

   ~~~java
   [root@node1 /]# memcached -p11211 -u nobody -d //-d表示以守护进程运行
   [root@node1 /]# netstat -luntp|grep 11211
   tcp        0      0 0.0.0.0:11211           0.0.0.0:*               LISTEN      31543/memcached     
   tcp6       0      0 :::11211                :::*                    LISTEN      31543/memcached     
   udp        0      0 0.0.0.0:11211           0.0.0.0:*                           31543/memcached     
   udp6       0      0 :::11211                :::*                                31543/memcached 
   ~~~

4. 修改nginx配置

   ~~~java
   [root@node1 conf.d]# pwd
   /etc/nginx/conf.d
   
   server {
       listen       80;
       server_name  localhost;
   
       #charset koi8-r;
       access_log  /var/log/nginx/log/host.access.log  main;
       
       location /hello {
           default_type 'text/plain';
           content_by_lua 'ngx.say("hello, lua")';
       }
    
       location /myip {
           default_type 'text/plain';
           content_by_lua '
               clientIP = ngx.req.get_headers()["x_forwarded_for"]
               ngx.say("IP:",clientIP)
               ';
       }
   
       location / {
           default_type "text/html"; 
           content_by_lua_file /opt/app/lua/dep.lua;
           #add_after_body "$http_x_forwarded_for";
       }
   
       location @server{
           proxy_pass http://127.0.0.1:9090;
       }
   
       location @server_test{
           proxy_pass http://127.0.0.1:8080;
       }
   
       error_page   500 502 503 504 404  /50x.html;
       location = /50x.html {
           root   /usr/share/nginx/html;
       }
   
   }
   ~~~

5. lua脚本

   ~~~lua
   clientIP = ngx.req.get_headers()["X-Real-IP"]
   if clientIP == nil then
       clientIP = ngx.req.get_headers()["x_forwarded_for"]
   end
   if clientIP == nil then
       clientIP = ngx.var.remote_addr
   end
       local memcached = require "resty.memcached" //lua调用memcached模块
       local memc, err = memcached:new()
       if not memc then
           ngx.say("failed to instantiate memc: ", err)
           return
       end
       local ok, err = memc:connect("127.0.0.1", 11211)
       if not ok then
           ngx.say("failed to connect: ", err)
           return
       end
       local res, flags, err = memc:get(clientIP)
       ngx.say("value key: ",res,clientIP)
       if err then
           ngx.say("failed to get clientIP ", err)
           return
       end
       if  res == "1" then
           ngx.exec("@server_test")
           return
       end
       ngx.exec("@server")
   ~~~

6. 安装lua调用memcached模块

   ~~~java
   wget https://github.com/agentzh/lua-resty-memcached/archive/v0.11.tar.gz
   tar -zxvf v0.11.tar.gz 
   cp -r lua-resty-memcached-0.11/lib/resty /usr/local/share/lua/5.1/ // 将当前库拷贝到系统指定的lua版本指定的库下
   ~~~

7. set memcached初始值

   ~~~java
   [root@node1 conf.d]# telnet 127.0.0.1 11211
   Trying 127.0.0.1...
   Connected to 127.0.0.1.
   Escape character is '^]'.
   set 192.168.x.xxx 0 0 1
   1
   STORED
   get 192.168.x.xxx
   VALUE 192.168.x.xxx 0 1
   1
   END
   quit
   Connection closed by foreign host.
   ~~~

8. 访问测试

   ~~~java
   http://192.168.x.xxx/java_test.jsp
   ~~~





