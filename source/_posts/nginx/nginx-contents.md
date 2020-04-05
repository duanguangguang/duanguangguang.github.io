---
title: nginx目录
date: 2020-03-28 14:31:33
categories: nginx
tags:
  - nginx
---

### 一、Nginx安装目录

通过yum安装其实都是装的一个个的rpm包，查找安装的rpm包：

~~~java
[root@node1 nginx]# rpm -ql nginx
~~~

<!-- more -->

| 路径                                        | 作用                                           |
| ------------------------------------------- | ---------------------------------------------- |
| /etc/logrotate.d/nginx                      | nginx日志轮转，用于logrotate服务的日志切割     |
| /etc/nginx/nginx.conf                       | nginx主配置文件，nginx启动主要读nginx.conf文件 |
| /etc/nginx/conf.d/default.conf              | 在没有变更情况下，默认的server加载的配置文件   |
| /etc/nginx/fastcgi_params                   | cgi配置相关                                    |
| /etc/nginx/uwscgi_params                    | cgi配置相关                                    |
| /etc/nginx/scgi_params                      | cgi配置相关                                    |
| /etc/nginx/koi-utf                          | 编码转换映射文件                               |
| /etc/nginx/koi-win                          | 编码转换映射文件                               |
| /etc/nginx/win-utf                          | 编码转换映射文件                               |
| /etc/nginx/mime.types                       | 设置http协议的Content-Type与扩展名对应关系     |
| /etc/sysconfig/nginx                        | 用于配置系统守护进程管理器管理方式             |
| /etc/sysconfig/nginx-debug                  | 同上                                           |
| /usr/lib/systemd/system/nginx-debug.service | 同上                                           |
| /usr/lib/systemd/system/nginx.service       | 同上                                           |
| /usr/lib64/nginx/modules                    | nginx模块目录                                  |
| /etc/nginx/modules                          | nginx模块目录                                  |
| /usr/sbin/nginx                             | nginx服务的启动管理的终端命令                  |
| /usr/sbin/nginx-debug                       | nginx服务的启动管理的终端命令                  |
| /usr/share/doc/...                          | nginx的手册和帮助文件                          |
| /usr/share/man/man8/nginx.8.gz              | nginx的手册和帮助文件                          |
| /var/chche/nginx                            | nginx的缓存目录                                |
| /var/log/nginx                              | nginx的日志目录                                |

1. etc：存放核心配置
2. usr
3. var

### 二、Nginx编译配置参数

~~~java
[root@node1 nginx]# nginx -V
nginx version: nginx/1.16.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
built with OpenSSL 1.0.2k-fips  26 Jan 2017
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
~~~

| 编译选项                                                  | 作用                                |
| --------------------------------------------------------- | ----------------------------------- |
| --prefix=/etc/nginx                                       | nginx主目录                         |
| --sbin-path=/usr/sbin/nginx                               | nginx的执行命令                     |
| --modules-path=/usr/lib64/nginx/modules                   | nginx的模块                         |
| --conf-path=/etc/nginx/nginx.conf                         | nginx的配置文件                     |
| --error-log-path=/var/log/nginx/error.log                 | nginx的错误日志                     |
| --http-log-path=/var/log/nginx/access.log                 | nginx的访问日志                     |
| --pid-path=/var/run/nginx.pid                             | nginx的pid文件                      |
| --lock-path=/var/run/nginx.lock                           | nginx锁路径                         |
| --http-client-body-temp-path=/var/cache/nginx/client_temp | nginx临时性文件                     |
| --http-proxy-temp-path=/var/cache/nginx/proxy_temp        | nginx临时性文件                     |
| --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp    | nginx临时性文件                     |
| --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp        | nginx临时性文件                     |
| --http-scgi-temp-path=/var/cache/nginx/scgi_temp          | nginx临时性文件                     |
| --user=nginx                                              | 设定nginx进程启动的用户             |
| --group=nginx                                             | 设定nginx进程启动的组用户           |
| --with-cc-opt='...'                                       | 设置额外的参数将被添加到CFLAGS变量  |
| --with-ld-opt='...'                                       | 设置附加的参数，链接系统库          |
| --with-...                                                | nginx所启用了哪些模块，后续模块介绍 |

### 三、默认配置语法

在`/etc/nginx`主目录下，默认配置文件nginx.conf：

~~~java
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
~~~

| 配置项           | 作用                        |
| ---------------- | --------------------------- |
| user             | 设置nginx服务的系统使用用户 |
| worker_processes | 工作进程数                  |
| error_log        | nginx的错误日志             |
| pid              | nginx服务启动的pid          |

- user

  默认nginx用户。

- worker_processes

  和nginx多进程有关，在IO多路复用启动多个进程来增大连接数的并发处理，一般和cpu个数保持一致即可。

| 模块   | 配置项             | 作用                         |
| ------ | ------------------ | ---------------------------- |
| events | worker_connections | 每个进程允许处理的最大连接数 |
|        | use                | 工作进程数                   |

- worker_connections

  最大可以调整到65535，一般调整到10000个就能满足企业需求。

- use

  定义使用的内核模型。

http模块介绍：

- include

  设置了所有的content-type的配置所在的地方。

- log_format

  定义了日志类型。

- access_log

  定义了访问日志路径。

- sendfile

  sendfile默认是打开的。

- keepalive_timeout

  设置了客户端和服务端之间的超时时间，单位：秒。

- include

  最后那个include是所有的子配置文件。

### 四、默认配置与默认站点启动

default_conf介绍：

```java
server {
    listen       80;
    server_name  localhost;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

- listen

  该server所监听的端口。

- server_name

  该server的服务名称。

- location

  一个server里面可以多个location，/表示访问根路径下所有子路径，比如访问首页路径等。

- root

  是location里面所返回的页面路径，如首页页面路径。

- index

  定义首页默认访问的页面。当index.html没有找到的话则访问index.htm。

- error_page

  错误页面同一定义为 /50x.html。