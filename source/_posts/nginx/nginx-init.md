---
title: nginx概述
date: 2020-03-28 14:26:10
categories: nginx
tags:
  - nginx
---

### 介绍

Nginx是一个开源且高性能、可靠的HTTP中间件、代理服务。那么什么是中间件服务呢？我们的网站后台往往存在很多的应用服务，对应的是我们的操作系统来驱动我们的硬件，应用于应用之间的互相调用，或者应用于操作系统之间的交互，在很多的应用存在的情况下，存在层次化的应用不够隔离，代码耦合程度高，所以我们需要中间件来为我们代理以及处理一些请求，让应用只负责业务的逻辑处理，这样就出现了中间件服务。

中间件可以起到与操作系统之间的调用，也可以分发给对应的应用进行相应的逻辑请求，这样整个网站承上启下，层次性的作用就会越发的好。下图是Nginx的中间件架构图示：

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-init\Nginx的中间件架构.png)

<!-- more -->

### 一、环境安装

1. 学习环境

   操作系统：

   - 版本>=7.0，位数X64
   - CentOS 7.2或redhat

2. 环境调试确认

   - 确认系统网络

     保证公网可以连通。

     ~~~java
     [root@node1 ~]# ping www.baidu.com
     ~~~

   - 确认yum可用

     软件安装包，以及nginx服务包都是通过yum源进行安装。

     ~~~java
     [root@node1 ~]# yum list|grep gcc
     ~~~

     ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-init\nginx确认yum.png)

   - 确认关闭iptables规则

     验证http服务在端口上进行屏蔽。

     查看iptables规则：

     ~~~java
     [root@node1 ~]# iptables -L
     ~~~

     关闭iptables规则：

     ~~~java
     [root@node1 ~]# iptables -F
     ~~~

     查看nat表中是否存在iptables规则，有的话也进行关闭：

     ~~~java
     [root@node1 ~]# iptables -t nat -L
     [root@node1 ~]# iptables -t nat -F
     ~~~

   - 确认停用selinux

     建议关闭selinux，会对服务正产运行以及请求进行安全性的规则屏蔽。

     查看selinux是否开启：

     ~~~java
     [root@node1 ~]# getenforce
     Enforcing
     ~~~

     关闭selinux：（临时）

     ~~~java
     [root@node1 ~]# setenforce 0
     [root@node1 ~]# getenforce
     Permissive
     ~~~

3. 两项安装

   - 基本库

     ~~~java
     [root@node1 ~]# yum -y install gcc gcc-c++ autoconf pcre pcre-devel make automake
     ~~~

   - 基本工具

     ~~~java
     [root@node1 ~]# yum -y install wget httpd-tools vim
     ~~~

4. 初始化目录

   定义代码应用，下载程序包，日志，备份等存储位置：

   ~~~java
   [root@node1 ~]# cd /opt
   [root@node1 ~]# mkdir app download logs work backup
   ~~~

### 二、常见的HTTP服务

1. HTTPD
   - Apache基金会

2. Nginx
3. LLS
   - 微软
4. GWS
   - Google

### 三、Nginx特性

1. IO多路复用epoll

   io复用图示：

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-init\IO复用图.png)

   io多路复用

   多个描述符的I/O操作都能在一个线程内并发交替地顺序完成，这里的`复用`指的是复用同一个线程。IO多路复用的实现方式select、poll、epoll。

   如何理解呢？举个例子吧！
   有A、B、C三个老师，他们都遇到一个难题，要帮助一个班级的学生解决课堂作业。
   老师A采用从第一排开始一个学生一个学生轮流解答的方式去回答问题，老师A浪费了很多时间，并且有的学生作业还没有完成呢，老师就来了，反反复复效率极慢。
   老师B是一个忍者，他发现老师A的方法行不通，于是他使用了影分身术，分身出好几个自己同一时间去帮好几个同学回答问题，最后还没回答完，老师B消耗光了能量累倒了。
   老师C比较精明，他告诉学生，谁完成了作业举手，有举手的同学他才去指导问题，他让学生主动发声，分开了“并发”。
   这个老师C就是Nginx。

   - select

     ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-init\select模式.png)

     - select模式采用线程遍历的模式，线性扫描效率低下
     - 能够监视文件描述符的数量存在最大限制，1024个

   - poll

   - epoll

     - 每当FD就绪，采用系统的回调函数之间将fd放入，效率更高
     - 无最大连接数限制

2. 轻量级

   - 功能模块少

     Nginx仅保留了HTTP需要的模块，其他都用插件的方式，后天添加。

   - 代码模块化

      更适合二次开发，如阿里巴巴Tengine。

3. CPU亲和（affinity）

   是一种把CPU核心和nginx工作进程绑定方式，把每个worker进程固定在一个cpu上执行，减少切换cpu的cache miss，获得更好的性能。

   nginx正是利用到他的cpu的亲和，来提升他的并发处理能力和对cpu的工作处理能力，减少不必要的额外性能损耗。

4. sendfile

   nginx在处理静态文件是非常有优势的，是因为nginx传输文件采用了sendfile的工作机制。以原来的http server文件传输为例，当我们需要请求一个文件的时候，需要进过操作系统的内核空间和用户空间，才能到达socket，通过socket再传递response给用户，对于操作系统而言，需要发生多次的切换。如下图：

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-init\传统sendfile.png)

   其实静态文件不需要经过操作系统用户空间逻辑性处理，直接通过内核空间处理，nginx正是用到了linux在2.2以后出来的这种`零拷贝`传输模式，文件传输只通过内核空间传递给socket，响应给用户。如下图：

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-init\nginx的sendfile.png)

### 四、Nginx安装

[Nginx官网下载](http://nginx.org/en/download.html )

1. nginx版本

   - Mainline version：开发版
   - Stable version：稳定版
   - Legacy version：历史版本

2. 安装

   新建nginx.repo文件

   ~~~java
   [root@node1 ~]# cd /etc/yum.repos.d/
   [root@node1 yum.repos.d]# vim nginx.repo
   ~~~

   复制官网的yum源：

   ~~~java
   [nginx-stable]
   name=nginx stable repo
   baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
   gpgcheck=1
   enabled=1
   gpgkey=https://nginx.org/keys/nginx_signing.key
   module_hotfixes=true
   ~~~

   查询可用的nginx版本：

   ~~~java
   [root@node1 yum.repos.d]# yum list | grep nginx
   ~~~

   安装nginx：

   ~~~java
   [root@node1 nginx]# yum install nginx
   ~~~

   查看nginx版本：

   ~~~java
   [root@node1 nginx]# nginx -v
   ~~~

   查看nginx编译参数：

   ~~~java
   [root@node1 nginx]# nginx -V
   nginx version: nginx/1.16.1
   built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
   built with OpenSSL 1.0.2k-fips  26 Jan 2017
   TLS SNI support enabled
   configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
   ~~~

   安装成功。



