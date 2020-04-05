---
title: nginx动静分离
date: 2020-04-04 11:46:37
categories: nginx
tags:
  - nginx
---

### 介绍

通过中间件将动态请求和静态请求分离。对服务端而言，分离了资源，减少了不必要的请求消耗，对客户端而言，减少请求延时。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-movement\动静分离.png)

<!-- more -->

### 一、动静分离场景

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-movement\动静分离场景.png)

1. 测试test_mysite.html文件

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
           url: "http://192.168.x.xxx:8080/java_test.jsp",
           success: function(data) {
               $("#get_data").html(data)
           },
           error: function() {  
               alert("fail!!!,请刷新再试!");  
           }  
       });  
   });  
   </script>  
   <body>  
       <h1>测试动静分离</h1>
       <img src="http://192.168.x.xxx/images/nginx.png"/>
       <div id="get_data"><div>
   </body>
   </html>  
   
   ~~~

2. jsp文件，动态生成随机数页面

   ~~~java
   <%@ page language="java" import="java.util.*" pageEncoding="utf-8"%>  
   <HTML>
       <HEAD>
           <TITLE>JSP Test Page</TITLE>
       </HEAD>
       <BODY>
           <%
               Random rand = new Random();
               out.println("<h1>Random number:</h1>");
               out.println(rand.nextInt(99)+100);
           %>
       </BODY>
   </HTML>
   ~~~

3. 动静分离配置

   ~~~java
   upstream java_api{
       server 192.168.x.xxx:8080; // 配置tomcat
   }
   server {
       listen       80;
       server_name  192.168.x.xxx node1;
   
       #charset koi8-r;
       access_log  /var/log/nginx/log/host.access.log  main;
       root /opt/app/code;  
   
       location ~ \.jsp$ {
           proxy_pass http://java_api;
           index  index.html index.htm;
       }
   
   
       location ~ \.(jpg|png|gif)$ {
           expires 1h;
           gzip on;
       }
   
       location /{
           index  index.html index.htm;
       }
   }
   ~~~

4. 重启tomcat

   ~~~java
   [root@node1 apache-tomcat-9.0.33]# cd bin
   [root@node1 bin]# sh catalina.sh stop
   Using CATALINA_BASE:   /opt/soft/tomcat/apache-tomcat-9.0.33
   Using CATALINA_HOME:   /opt/soft/tomcat/apache-tomcat-9.0.33
   Using CATALINA_TMPDIR: /opt/soft/tomcat/apache-tomcat-9.0.33/temp
   Using JRE_HOME:        /home/duan/myhome/soft/jdk/jdk1.8.0_231/jre
   Using CLASSPATH:       /opt/soft/tomcat/apache-tomcat-9.0.33/bin/bootstrap.jar:/opt/soft/tomcat/apache-tomcat-9.0.33/bin/tomcat-juli.jar
   [root@node1 bin]# ps -aux | grep java
   root       5255  0.0  0.0 112728   972 pts/0    R+   16:39   0:00 grep --color=auto java
   [root@node1 bin]# sh catalina.sh start;tail -f ../logs/catalina.out // 打印启动日志
   [root@node1 bin]# netstat -luntp|grep 8080
   tcp6       0      0 :::8080              :::*          LISTEN    5295/java    
   ~~~

5. 重载nginx配置 `nginx -c /etc/nginx/nginx.conf`

   ~~~java
   [root@node1 nginx]# nginx -tc /etc/nginx/nginx.conf 
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   [root@node1 nginx]# nginx -s reload -c /etc/nginx/nginx.conf 
   ~~~

6. 访问测试

   ~~~java
   http://192.168.x.xxx/test_mysite.html
   ~~~

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-movement\测试动静分离.png)

