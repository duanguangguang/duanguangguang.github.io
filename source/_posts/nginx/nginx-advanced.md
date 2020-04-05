---
title: nginx高级模块
date: 2020-04-04 11:47:13
categories: nginx
tags:
  - nginx
---

### 介绍

介绍Nginx新的比较常用的模块。

<!-- more -->

### 一、secure_link_module模块

作用：防盗链的补充及完善。

1. 指定并允许检查请求的链接的真实性以及保护资源免遭未经授权的访问
2. 限制链接生效周期

该模块可以实现一些比较精度的高级的验证，利用的就是后端加密的机制返回给前端，实现加密数字签名的验证方式，保证请求链接的真实性。

配置语法：

~~~java
Syntax:	secure_link expression;
Default: --
Context: http,server,location

Syntax:	secure_link_md5 expression;
Default: --
Context: http,server,location
~~~

验证机制流程：

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-advanced\验证机制流程.png)

场景演示：

1. 准备一个下载文件file.img

   ~~~java
   [root@node1 download]# ls
   test.img.gz
   ~~~

2. 编写MD5加密shell脚本

   ~~~java
   #!/bin/sh
   #
   #Auth:Duan
   servername="node1"
   download_file="/download/test.img.gz"
   time_num=$(date -d "2020-04-05 00:00:00" +%s)
   secret_num="node1"
   
   res=$(echo -n "${time_num}${download_file} ${secret_num}"|openssl md5 -binary | openssl base64 | tr +/ -_ | tr -d =)
   
   echo "http://${servername}${download_file}?md5=${res}&expires=${time_num}"
   ~~~

   生成下着链接：

   ~~~java
   [root@node1 work]# sh md5url.sh 
   http://node1/download/test.img.gz?md5=XmTCRx0yExVUKQOSlbI2AA&expires=1586016000
   ~~~

3. 配置nginx

   ~~~java
   server {
       listen       80;
       server_name  localhost;
   
       #charset koi8-r;
       #access_log  /var/log/nginx/log/host.access.log  main;
       root /opt/app/code;
   
       location / {
           secure_link $arg_md5,$arg_expires;
           secure_link_md5 "$secure_link_expires$uri node1";
   
           if ($secure_link = "") {
               return 403;
           }
   
           if ($secure_link = "0") {
               return 410;
           }
       }
   }
   ~~~

4. 访问测试

   ~~~java
   http://node1/download/test.img.gz?md5=XmTCRx0yExVUKQOSlbI2AA&expires=1586016000
   ~~~

   下载成功。

### 二、Geoip读取地域信息模块

基于IP地址匹配MaxMind GeoIP二进制文件，读取IP所在地域信息。默认安装没有安装该模块，所以需要先安装该模块：

~~~java
yum install nginx-module-geoip
~~~

使用场景：

1. 区别国内外作HTTP访问规则
2. 区别国内城市地域作HTTP访问规则

geoip数据文件下载地址：

~~~java
wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCountry/GeoIP.dat.gz
wget http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz
~~~

nginx配置：

~~~java
geoip_country /etc/nginx/geoip/GeoIP.dat;
geoip_city /etc/nginx/geoip/GeoLiteCity.dat;
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        if ($geoip_country_code != CN) {
            return 403;
        }
        root   /opt/app/code/html;
        index  index.html index.htm;
    }

   location /myip {
        default_type text/plain;
        return 200 "$remote_addr $geoip_country_name $geoip_country_code $geoip_city";
   }
}
~~~

