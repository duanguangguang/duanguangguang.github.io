---
title: nginx常见问题
date: 2020-04-05 23:49:09
categories: nginx
tags:
  - nginx
---

### 介绍

介绍nginx常见问题。

<!-- more -->

### 一、server_name优先级

~~~java
server {
    listen       80;
    server_name  testserver1 node1;
    
    location / {
        root   /opt/app/code1;
        index  index.html index.htm;
    }
}

server {
    listen       80;
    server_name  testserver2 node2;
    
    location / {
        root   /opt/app/code2;
        index  index.html index.htm;
    }
}
~~~

比如说两个nginx配置文件如上，一个是test1.conf，一个是test2.conf，配置相同的server_name，即使产生了冲突，nginx的启动也不会有问题。

对于多个虚拟主机，配置有相同的server_name的话，只是优先读取先读取到的配置（test_server1.conf）。对于IP配置也相同。

### 二、location优先级

location匹配的优先级从上往下是：优先匹配最精准的。

- `=` 进行普通字符精确匹配，完全匹配（匹配到则不继续查找匹配）
- `^~` 表示普通字符匹配，使用前缀匹配（匹配到则不继续查找匹配）
- `~\~*` 表示执行一个正则匹配 （匹配到则继续查找更精确的匹配）

~~~java
location = /code1/ {
        rewrite ^(.*)$ /code1/index.html break;
    }
    location ~ /code.* {
        rewrite ^(.*)$ /code3/index.html break;
    }
    location ^~ /code {
        rewrite ^(.*)$ /code2/index.html break;
    }
~~~

### 三、try_files使用

作用：按顺序检查文件是否存在。

~~~java
location / {
    try_files $uri $uri//index.html
}
~~~

- 首先查看uri内容本地是否存在，存在的话则解析返回给用户
- 没有的话将请求加/，表示将请求进行重定向处理，寻找这个路径下有没有文件
- 如果还是没有，则交给index.html来处理

~~~java
location / {
    root /opt/app/code/cache;
    try_files /cache $uri @java_page;
}


location @java_page{
    proxy_pass http://127.0.0.1:9090;
} 
~~~

~~~java
http://192.168.x.xxx/index.html
~~~

- 先请求/opt/app/code/cache这个路径下文件是否存在
- 不存在则请求@java_page对应的location，转发到http://127.0.0.1:9090进行处理

### 四、alias和root区别

1. root

   ~~~java
   location /request_path/image/ {
       root /local_path/image;
   }
   
   // 请求：http://www.imooc.com/request_path/image/cat.png
   /local_path/image/request_path/image/cat.png
   ~~~

2. alias

   ~~~java
   location /request_path/image/ {
       alias /local_path/image;
   }
   
   // 请求：http://www.imooc.com/request_path/image/cat.png
   /local_path/image/cat.png
   ~~~

### 五、获取用户真实IP

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-common\获取用户真实IP.png)

### 六、Nginx常见错误码

1. 用户上传文件限制：`client_max_body_size`

   Nginx：413 Request Entity Too Large。

2. 后端服务无响应

   502 bad gateway。

3. 后端服务响应超时

   504 Gateway Time-out。默认60s。

4. 403

   访问被拒绝。

5. 404

   文件没找到。

6. 400

   请求参数错误。

