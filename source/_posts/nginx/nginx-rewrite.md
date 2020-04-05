---
title: nginx Rewrite规则
date: 2020-04-04 11:46:49
categories: nginx
tags:
  - nginx
---

### 介绍

在nginx的rewrite规则里边，可以实现对url的重写以及对匹配的url的重定向，匹配依赖于正则表达式。使用场景：

1. URL访问跳转，支持开发设计

   页面跳转，兼容性支持，展示效果等

2. SEO优化

   对于前端页面的优化，对于google或者百度，他们的搜索引擎的优化，搜索效率排名依赖于搜索路径的，如果我们的路径过于复杂，不符合搜索规则的话，会影响搜索引擎的录入，这时就需要nginx对后台的接口进行伪静态的改写，以符合对应的搜索引擎的搜索规范，这个就是seo的优化。

3. 运维维护

   后台维护，流量转发等。

4. 安全

   可以实现伪静态。

<!-- more -->

### 一、配置语法

~~~java
Syntax:	rewrite regex replacement[flag];
Default: --
Context: server,location,if
~~~

- regex

  正则表达式，用于匹配需要进行改写的url或者路径。

- replacement

  目标要替换成的url或者路径。

- flag

  标识。

实例例：`rewrite ^(.*)$/pages/maintain.html break`，将所有的请求都重定向到/pages/maintain.html页面。

### 二、正则表达式

| 规则  | 描述                                                      |
| ----- | --------------------------------------------------------- |
| .     | 匹配除换行符以外的任意字符                                |
| ?     | 重复0次或1次                                              |
| +     | 重复1次或更多次                                           |
| *     | 贪婪匹配                                                  |
| \d    | 匹配数字                                                  |
|       |                                                           |
| ^     | 匹配字符串开头                                            |
| $     | 匹配字符串结尾                                            |
| {n}   | 重复n次                                                   |
| {n,}  | 重复n次或更多次                                           |
| [c]   | 匹配单个字符c                                             |
| [a-z] | 匹配a-z小写字符任意一个                                   |
|       |                                                           |
| \     | 转义字符（rewrite index\.php$/pages/maintain.html break） |
| ( )   | 用于匹配括号之间的内容，通过\$1、$2调用                   |

~~~java
if ($http_user_agent ~ MSIE) {
    rewrite ^(.*)$ /msie/$1 break; //(.*)中的内容会全部放到$1
}
~~~

终端测试命令：`pcretest`。

~~~java
[root@node1 pcre-8.44]# pcretest
PCRE version 8.44 2020-02-12

  re> /(\d+)\.(\d+)\.(\d+)\.(\d+)/
data> 192.168.1.1
 0: 192.168.1.1
 1: 192
 2: 168
 3: 1
 4: 1
data> 
~~~

### 三、flag

标记rewrite规则对应的类型，常见的有：

| flag      | 描述                                        |
| --------- | ------------------------------------------- |
| last      | 停止rewrite检测                             |
| break     | 停止rewrite检测                             |
| redirect  | 返回302临时重定向，地址栏会显示跳转后的地址 |
| permanent | 返回301永久重定向，地址栏会显示跳转后的地址 |

last与break区别：

- break访问到该级location之后，找到对应的rewrite跳转，实现该跳转回到root程序目录下查找对应的访问路径（）是否存在（下面示例没有配置），没有找到会返回404
- last匹配到location之后，会实现一次跳转，last会新建一个请求，相当于会重新请求一次服务端，请求的地址变成了新的匹配到的rewrite指定的地址（本质是一次请求）

示例配置：

~~~java
server {
    listen 80 default_server;
    server_name 192.168.x.xxx node1;

    access_log  /var/log/nginx/log/host.access.log  main;
     
    root /opt/app/code; 
    location ~ ^/break {
        rewrite ^/break /test/ break; // 去root /opt/app/code下找test路径
    } 
 
    location ~ ^/last {
         rewrite ^/last /test/ last; // last用匹配到的rewrite的test路径请求，相当于下面的test直接请求
    }    
 
    location /test/ {
       default_type application/json;
       return 200 '{"status":"success"}';
    }
}
~~~

### 四、redirect与permanent区别

1. redirect与last区别

   - last

     ~~~java
     location ~ ^/last {
              rewrite ^/last /test/ last;
         }    
     ~~~

     访问测试：

     ~~~java
     [root@node1 conf.d]# curl -vL 192.168.x.xxx/last
     ~~~

     结果：

     ~~~java
     [root@node1 conf.d]# curl -vL 192.168.x.xxx/last
     * About to connect() to 192.168.x.xxx port 80 (#0)
     *   Trying 192.168.x.xxx...
     * Connected to 192.168.x.xxx (192.168.x.xxx) port 80 (#0)
     > GET /last HTTP/1.1
     > User-Agent: curl/7.29.0
     > Host: 192.168.x.xxx
     > Accept: */*
     > 
     < HTTP/1.1 200 OK
     < Server: nginx/1.16.1
     < Date: Sat, 04 Apr 2020 12:00:32 GMT
     < Content-Type: application/json
     < Content-Length: 20
     < Connection: keep-alive
     < 
     * Connection #0 to host 192.168.x.xxx left intact
     ~~~

     可以看到一次请求，返回200状态码。

   - redirect

     ~~~java
     location ~ ^/last {
              rewrite ^/last /test/ redirect;
         }    
     ~~~

     访问测试：

     ~~~java
     [root@node1 conf.d]# curl -vL 192.168.x.xxx/last
     ~~~

     结果：

     ~~~java
     [root@node1 conf.d]# curl -vL 192.168.x.xxx/last
     * About to connect() to 192.168.x.xxx port 80 (#0)
     *   Trying 192.168.x.xxx...
     * Connected to 192.168.x.xxx (192.168.x.xxx) port 80 (#0)
     > GET /last HTTP/1.1
     > User-Agent: curl/7.29.0
     > Host: 192.168.x.xxx
     > Accept: */*
     > 
     < HTTP/1.1 302 Moved Temporarily
     < Server: nginx/1.16.1
     < Date: Sat, 04 Apr 2020 12:05:04 GMT
     < Content-Type: text/html
     < Content-Length: 145
     < Location: http://192.168.x.xxx/test/
     < Connection: keep-alive
     < 
     * Ignoring the response-body
     * Connection #0 to host 192.168.x.xxx left intact
     * Issue another request to this URL: 'http://192.168.x.xxx/test/'
     * Found bundle for host 192.168.x.xxx: 0x2306ee0
     * Re-using existing connection! (#0) with host 192.168.x.xxx
     * Connected to 192.168.x.xxx (192.168.x.xxx) port 80 (#0)
     > GET /test/ HTTP/1.1
     > User-Agent: curl/7.29.0
     > Host: 192.168.x.xxx
     > Accept: */*
     > 
     < HTTP/1.1 200 OK
     < Server: nginx/1.16.1
     < Date: Sat, 04 Apr 2020 12:05:04 GMT
     < Content-Type: application/json
     < Content-Length: 20
     < Connection: keep-alive
     < 
     * Connection #0 to host 192.168.x.xxx left intact
     ~~~

     可以看到请求了两次，返回200之前先返回了302状态码。`Location: http://192.168.x.xxx/test/`

     表示接下来要请求的地址。

2. redirect与permanent区别

   - redirect

     ~~~java
     location ~ ^/imooc {
              #rewrite ^/imooc http://www.imooc.com/ permanent;
              rewrite ^/imooc http://www.imooc.com/ redirect;
         }
     ~~~

     访问测试：

     ~~~java
     http://192.168.x.xxx/imooc
     ~~~

     跳转正常.将nginx关闭，再次访问结果：无法连接。

   - permanent

     ~~~java
     location ~ ^/imooc {
              rewrite ^/imooc http://www.imooc.com/ permanent;
              #rewrite ^/imooc http://www.imooc.com/ redirect;
         }
     ~~~

     访问测试：

     ~~~java
     http://192.168.x.xxx/imooc
     ~~~

     跳转正常。将nginx关闭，再次访问结果：同样跳转正常。

### 五、rewrite规则场景

1. 请求链接目录归级

   ~~~java
   rewrite ^/course-(\d+)-(\d+)-(\d+)\.html$ /course/$1/$2/course_$3.html break;
           if ($http_user_agent ~* Chrome) {
               rewrite ^/nginx http://coding.imooc.com/class/121.html redirect;
           } 
   ~~~

2. 匹配文件，不存转发到百度查询

   ~~~java
   if (!-f $request_filename) {
               rewrite ^/(.*)$ http://www.baidu.com/$1 redirect;
           }
   ~~~

### 六、rewrite规则优先级

- 执行server块的rewrite执行
- 执行location匹配
- 执行选定的location中的rewrite

优雅的rewrite规则书写：

apache下书写：

~~~java
RewriteCond %{HTTP_HOST} nginx.org
RewriteRule (.*)
~~~

把主机名对应nginx,org进行转发。对应在nginx中写法：

~~~java
server {
    listen 80;
    server_name www.nginx.org nginx.org;
    if ($http_host = nginx.org) {
        rewrite (.*) http://www.nginx.org$1;
    }
}
~~~

官方推荐写法：

~~~javaa
server {
    listen 80;
    server_name nginx.org;
    rewrite ^ http://www.nginx.org$request_uri?;
}

server {
    listen 80;
    server_name nginx.org;
    ...
}
~~~

