---
title: nginx作为缓存服务
date: 2020-03-31 23:23:56
categories: nginx
tags:
  - nginx
---

### 介绍

缓存类型：

- 客户端缓存
- 代理缓存
- 服务端缓存

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-cache\缓存流程.png)

<!-- more -->

### 一、配置语法

~~~java
Syntax:	proxy_cache_path path[levels=levels][use_temp_path=on|off]
keys_zone=name:size[inactive=time][max_size=size][manager_files=number]
[manager_sleep=time][manager_threshold=time][loader_files=number]
[loader_sleep=time][loder_threshold=time][purger=on|off]
[purger_files=number][purger_sleep=time][pruger_threshold=time];
Default: --
Context: http
~~~

proxy_cache配置：

~~~java
Syntax:	proxy_cache zone|off;
Default: proxy_cache off;
Context: http,server,location
~~~

缓存过期周期配置：

~~~java
Syntax:	proxy_cache_valid[code...]time;
Default: --;
Context: http,server,location
~~~

缓存维度配置：

~~~java
Syntax:	proxy_cache_key string;
Default: proxy_cache_key $scheme$proxy_host$request_uri;
Context: http,server,location
~~~

### 二、场景配置演示

~~~java
upstream cache {
        server 192.168.x.xxx:8001;
        server 192.168.x.xxx:8002;
        server 192.168.x.xxx:8003;
    }

    proxy_cache_path /opt/app/cache levels=1:2 keys_zone=test_cache:10m max_size=10g inactive=60m use_temp_path=off;

server {
    listen       80;
    server_name  localhost node1;

    #charset koi8-r;
    access_log  /var/log/nginx/test_proxy.access.log  main;

    
    location / {
        #proxy_cache off;
        proxy_cache test_cache;
        proxy_pass http://cache;
        proxy_cache_valid 200 304 12h;
        proxy_cache_valid any 10m;
        proxy_cache_key $host$uri$is_args$args;
        add_header  Nginx-Cache "$upstream_cache_status";  
        
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        include proxy_params;
    }
}
~~~

- proxy_cache_path

  定义好目录空间大小以及名字，用来存放对应的缓存文件的。

- levels

  表示缓存的文件按照目录进行分级。1:2按照两层目录分级

- keys_zone

  开辟的zone空间的名字。10m：开辟空间的大小。一般1m可以存放8K个key。

- max_size

  目录控制最大大小。当空间已满，nginx会使用自己的淘汰机制，把一些不常用到的淘汰掉。

- inactive

  在60分钟内，没有访问过的文件被清理掉。

- use_temp_path

  用来存放临时文件的，建议关闭。打开会另外建立一个目录，和cache目录两个目录，在更新缓存时会出现一些性能损耗。

- proxy_cache_valid 200 304 12h

  表示200,304的头信息过期时间是12个小时。

- proxy_cache_valid any 10m

  其他状态的头信息是10分钟过期。

- proxy_next_upstream

  如果后端服务器出现了50*不正常的头信息返回，就跳过该服务器，访问下一台服务器。

### 三、清除指定缓存

1. 方式一

   rm -rf缓存目录。

2. 方式二

   第三方扩展模块ngx_cache_purge。

### 四、部分页面不缓存

~~~java
Syntax:	proxy_no_cache string;
Default: --;
Context: http,server,location
~~~

场景配置：

~~~java
if ($request_uri ~ ^/(url3|logon|register|password\/reset)) { //配置url3不缓存
    set $cookie_nocache 1;
}

location / {
        proxy_cache test_cache;
        proxy_no_cache $cookie_nocache $arg_nocache $arg_comment;
        proxy_no_cache $http_pragma $http_authorization;
        add_header  Nginx-Cache "$upstream_cache_status";  
        
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        include proxy_params;
    }
~~~

### 五、分片请求

大文件分片请求：

~~~java
Syntax:	slice size;
Default: slice 0;
Context: http,server,location
~~~

size表示对大文件切割成多大的小文件碎片。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-cache\分片.png)

优势：

每个字请求收到的数据都会形成一个独立文件，一个请求断了，其他请求不受影响。

缺点：

当文件很大或者slice很小的时候，可能会导致文件描述符耗尽等情况。