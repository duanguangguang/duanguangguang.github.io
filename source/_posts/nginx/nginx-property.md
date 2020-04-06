---
title: nginx性能优化
date: 2020-04-05 23:49:42
categories: nginx
tags:
  - nginx
---

### 介绍

性能优化需要考虑的点：

1. 当前系统结构瓶颈

   观察指标（top，日志）、压力测试（ab压测工具）。

2. 了解业务模式

   接口业务类型（秒杀，抢购）、系统层次化结构。

3. 性能与安全

   设计防火墙功能。

<!-- more -->

### 一、ab压测工具

1. 安装

   ~~~java
   [root@node1 conf.d]# yum install httpd-tools
   ~~~

2. 使用

   ~~~java
   ab -n 2000 -c 2 http://127.0.0.1/
   ~~~

   - -n：总的请求书
   - -c：并发数
   - -k：是否开启长连接

ab工具演示：

1. nginx配置

   ~~~java
   server {
       listen       80;
       server_name  192.168.1.104 node1;
   
       #charset koi8-r;
       #access_log  /var/log/nginx/log/host.access.log  main;
       #root   /opt/app;
       
       location / {
           root /opt/app/code/cache/;
           try_files /cache $uri @java_page;
       }
   
       location @java_page{
           proxy_pass http://192.168.1.104:9090;
       } 
   
       #error_page  404              /404.html;
   
       # redirect server error pages to the static page /50x.html
       #
       error_page   500 502 503 504 404  /50x.html;
       location = /50x.html {
           root   /usr/share/nginx/html;
       }
   }
   ~~~

2. 压测结果

   ~~~java
   [root@node1 conf.d]# ab -n 2000 -c 2 http://192.168.1.104/admin.html
   This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
   Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
   Licensed to The Apache Software Foundation, http://www.apache.org/
   
   Benchmarking 192.168.1.104 (be patient)
   Completed 200 requests // 输出的进度，完成了多少个
   Completed 400 requests
   Completed 600 requests
   Completed 800 requests
   Completed 1000 requests
   Completed 1200 requests
   Completed 1400 requests
   Completed 1600 requests
   Completed 1800 requests
   Completed 2000 requests
   Finished 2000 requests
   
   
   Server Software:        nginx/1.16.1 // 工具探测到的软件信息
   Server Hostname:        192.168.1.104
   Server Port:            80
   
   Document Path:          /admin.html // 文件的路径
   Document Length:        143 bytes // body，该文件返回的大小，143字节
   
   Concurrency Level:      2 // 并发级别
   Time taken for tests:   0.652 seconds // 花费的总时间
   Complete requests:      2000 // 总的完成的请求数
   Failed requests:        0 // 失败的请求数
   Write errors:           0
   Total transferred:      750000 bytes
   HTML transferred:       286000 bytes
   Requests per second:    3065.32 [#/sec] (mean) // 每秒所能处理的请求QPS(总请求/总时间)
   Time per request:       0.652 [ms] (mean) // 一个请求完成的时间，客户端访问服务端
   Time per request:       0.326 [ms] (mean, across all concurrent requests) // 服务端处理一个请求的时间，不包括客户端与服务端网络传输的时间
   Transfer rate:          1122.55 [Kbytes/sec] received // 速率，判断网络是否存在瓶颈
   
   Connection Times (ms)
                 min  mean[+/-sd] median   max
   Connect:        0    0   0.2      0       4
   Processing:     0    0   0.3      0       4
   Waiting:        0    0   0.3      0       3
   Total:          0    1   0.4      1       5
   
   Percentage of the requests served within a certain time (ms)
     50%      1
     66%      1
     75%      1
     80%      1
     90%      1
     95%      1
     98%      2
     99%      2
    100%      5 (longest request)
   ~~~

3. 测试动态接口

   tomcat下部署jsp文件：

   ~~~java
   <%@ page language="java" import="java.util.*" pageEncoding="utf-8"%>  
   <HTML>
       <HEAD>
           <TITLE>JSP Test Page</TITLE>
       </HEAD>
       <BODY>
           <%
               Thread.sleep(5000);
               Random rand = new Random();
               out.println("<h1>Random number:</h1>");
               out.println(rand.nextInt(99)+100);
           %>
       </BODY>
   </HTML>
   ~~~

4. 启动tomcat

   ~~~java
   [root@node1 bin]# sh catalina.sh start;tail -f ../logs/catalina.out
   [root@node1 bin]# netstat -luntp|grep java
   tcp6       0      0 :::9090                 :::*                    LISTEN      6410/java           
   tcp6       0      0 127.0.0.1:9005          :::*                    LISTEN      6410/java           
   tcp6       0      0 127.0.0.1:9009          :::*                    LISTEN      6410/java
   ~~~

5. 单个请求测试

   ~~~java
   [root@node1 bin]# curl -I http://127.0.0.1/sleepjava.jsp
   HTTP/1.1 200 
   Server: nginx/1.16.1
   Date: Mon, 06 Apr 2020 08:23:23 GMT
   Content-Type: text/html;charset=utf-8
   Connection: keep-alive
   Set-Cookie: JSESSIONID=26F706E6ED964C1B9941128321DE4A4F; Path=/; HttpOnly
   ~~~

6. ab测试20的请求

   ~~~java
   [root@node1 bin]# ab -n 20 -c 2 http://127.0.0.1/sleepjava.jsp
   This is ApacheBench, Version 2.3 <$Revision: 1430300 $>
   Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
   Licensed to The Apache Software Foundation, http://www.apache.org/
   
   Benchmarking 127.0.0.1 (be patient).....done
   
   
   Server Software:        nginx/1.16.1
   Server Hostname:        127.0.0.1
   Server Port:            80
   
   Document Path:          /sleepjava.jsp
   Document Length:        147 bytes
   
   Concurrency Level:      2
   Time taken for tests:   55.117 seconds
   Complete requests:      20
   Failed requests:        0
   Write errors:           0
   Total transferred:      7540 bytes
   HTML transferred:       2940 bytes
   Requests per second:    0.36 [#/sec] (mean)
   Time per request:       5511.728 [ms] (mean)
   Time per request:       2755.864 [ms] (mean, across all concurrent requests)
   Transfer rate:          0.13 [Kbytes/sec] received
   
   Connection Times (ms)
                 min  mean[+/-sd] median   max
   Connect:        0    0   0.2      0       1
   Processing:  5004 5011   5.4   5008    5020
   Waiting:     5004 5010   5.3   5008    5020
   Total:       5004 5011   5.5   5008    5020
   
   Percentage of the requests served within a certain time (ms)
     50%   5008
     66%   5011
     75%   5016
     80%   5020
     90%   5020
     95%   5020
     98%   5020
     99%   5020
    100%   5020 (longest request)
   ~~~

### 二、系统与Nginx性能优化

1. 网络

   对于网络而言，我们要关注网络的流量是否有丢包。

2. 系统

   关注硬件有没有磁盘的损坏，磁盘的速率以及系统负载饱和，内存使用率以及系统的稳定性。

3. 服务

   更主要的就是服务本身，Nginx本身连接的优化以及内核性能的优化以及http请求的优化。

4. 程序

   接口性能，处理速度，程序的执行效率。

5. 数据库、底层服务

   分布式存储以及其他一些相关的服务的性能。

### 三、文件句柄设置

linux\Unix一切皆文件，文件句柄就是一个索引。默认操作系统会设置为1024个。对于nginx来说就太小了。

设置方式：系统全局性修改，用户局部性修改，进程局部性修改。

~~~java
[root@node1 /]# cd /etc/security/
[root@node1 security]# ls
limits.conf 

root soft nofile 65535 // 用户
root hard nofile 65535
*    soft nofile 25535 // 全局
*    hard nofile 25535
~~~

- soft：发提醒，操作系统不会强制的限制
- hard：操作系统会采取机制进行限制
- 一般设置1W就足够了

针对进程设置：

~~~java
[root@node1 nginx]# vim nginx.conf

worker_rlimit_nofile 35535; //进程
~~~

### 四、CPU亲和配置

CPU亲和，让进程通常不会在处理器之间频繁迁移进程，进程迁移频率小，减少性能损耗。

查看系统当前有多少个CPU：

~~~java
[root@node1 nginx]# cat /proc/cpuinfo|grep "physical id"|sort|uniq|wc -l
1
~~~

查看CPU核心：

~~~java
[root@node1 nginx]# cat /proc/cpuinfo|grep "cpu cores"|uniq
cpu cores	: 1
~~~

查看cpu总核心：

~~~java
[root@node1 nginx]# cat /proc/cpuinfo|grep "processor"|wc -l
1
~~~

CPU亲和配置：nginx.conf，以16核心为例：

~~~java
user  nginx;
worker_processes  16; // 当前启动的worker进程，官方建议与CPU核心一致
#worker_cpu_affinity 0000000000000010 0000000000000010 0000000000000100 0000000000001000 0000000000010000 0000000000100000 0000000001000000 0000000010000000 0000000100000000 0000001000000000 0000010000000000 0000100000000000 0001000000000000 0010000000000000 0100000000000000 1000000000000000;
#worker_cpu_affinity 1010101010101010 0101010101010101;
worker_cpu_affinity auto;
~~~

- worker_cpu_affinity：表示当前是哪些CPU可以用的（worker_processes 2）

  第一个使用1,3,5,7,9...

查看当前Nginx所使用的CPU是哪一个：

~~~java
[root@node1 nginx]# ps -eo pid,args,psr | grep [n]ginx
  2662 nginx: master process nginx   0 //虚拟机配置只有一个核心
  6233 nginx: worker process         0
~~~

- ps -eo输出nginx这个进程
- pid：nginx的pid
- args：nginx的进程名
- psr：nginx使用的cpu是哪一个

### 五、Nginx通过配置优化

~~~java
user  nginx; // 建议配置普通用户nginx
worker_processes  16;
#worker_cpu_affinity 0000000000000010 0000000000000010 0000000000000100 0000000000001000 0000000000010000 0000000000100000 0000000001000000 0000000010000000 0000000100000000 0000001000000000 0000010000000000 0000100000000000 0001000000000000 0010000000000000 0100000000000000 1000000000000000;
#worker_cpu_affinity 1010101010101010 0101010101010101;
worker_cpu_affinity auto;


error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

worker_rlimit_nofile 35535;

events {
    use epoll;
    worker_connections  10240; //每个worker进程处理的个数
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    #######
    #Charset
    charset utf-8;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" "$request_uri"';

    access_log  /var/log/nginx/access.log  main;
    
    #######
    #Core modlue
    sendfile        on;
    #tcp_nopush     on;
    #tcp_nodeny     on;
    keepalive_timeout  65;
    
    ########
    #Gzip module
    gzip  on; // gzip配置成全局
    gzip_disable "MSIE [1-6]\.";  // IE6浏览器不支持一些gzip压缩，对于IE1-6配置不进行gzip压缩
    gzip_http_version 1.1; 
    
    ########
    #Virtal Server
    include /etc/nginx/conf.d/*.conf;
}
~~~

