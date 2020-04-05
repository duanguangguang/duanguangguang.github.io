---
title: nginx请求限制
date: 2020-03-28 14:32:39
categories: nginx
tags:
  - nginx
---

### 介绍

Nginx的请求限制分类：

- 连接频率限制

  `limit_conn_module`。

- 请求频率限制

  `limit_req_module`。

HTTP协议的连接与请求：

| HTTP协议版本 | 连接关系        |
| ------------ | --------------- |
| HTTP1.0      | TCP不能复用     |
| HTTP1.1      | 顺序性TCP复用   |
| HTTP2.0      | 多路复用TCP复用 |

<!-- more -->

HTTP请求建立在一次TCP连接基础上，一次TCP请求至少产生一次HTTP请求。

### 一、连接限制

配置语法：

~~~java
Syntax:	limit_conn_zone key zone=name:size;
Default: --
Context: http
~~~

- key：限制的key
- zone：限制空间
- name：限制空间的名字，为了limit_conn来调用的
- size：申请空间的大小，单位是：M

~~~java
Syntax:	limit_conn zone number;
Default: --
Context: http,server,location
~~~

- zone：就是需要调用的申请的zone的name
- number：表示并发的限制，个数

### 二、请求限制

配置语法：

~~~java
Syntax:	limit_req_zone key zone=name:size rate=rate;
Default: --
Context: http
~~~

- rate：速率，单位：秒（每秒几个）

~~~java
Syntax:	limit_req zone name [burst=number][nodelay];
Default: --
Context: http,server,location
~~~

- burst：表示的是客户端在超过了指定的速率后，配置的遗留的请求数（配置为3）放到下一秒去执行，延迟响应，对客户端起到一个限速的作用
- nodelay：超过3个的直接返回503

### 三、请求限制演示

1. 配置修改

   ~~~java
   limit_conn_zone $binary_remote_addr zone=conn_zone:1m;
   limit_req_zone $binary_remote_addr zone=req_zone:1m rate=1r/s;
   server {
       location / {
       root   /opt/app/code;
   	#limit_conn conn_zone 1;
   	#limit_req zone=req_zone burst=3 nodelay;
   	#limit_req zone=req_zone burst=3;
   	limit_req zone=req_zone;
       index  index.html index.htm;
       }
   }
   ~~~

2. 服务重载

   ~~~java
   root@node1 conf.d]# nginx -tc /etc/nginx/nginx.conf 
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   [root@node1 conf.d]# nginx -s reload -c /etc/nginx/nginx.conf
   ~~~

3. 压力测试

   ~~~java
   [root@node1 ~]# ab -n 20 -c 10 http://192.168.x.xxx/red.html
   ~~~

   - -n：发起的总请求数20个
   - -c：同时并发请求10个

   ~~~java
   Server Software:        nginx/1.16.1
   Server Hostname:        192.168.x.xxx
   Server Port:            80
   
   Document Path:          /red.html
   Document Length:        127 bytes
   
   Concurrency Level:      10
   Time taken for tests:   0.017 seconds
   Complete requests:      20 // 完成20个请求
   Failed requests:        19
      (Connect: 0, Receive: 0, Length: 19, Exceptions: 0)
   Write errors:           0
   Non-2xx responses:      19 // 19个请求返回非200状态
   Total transferred:      13431 bytes
   HTML transferred:       9513 bytes
   Requests per second:    1153.34 [#/sec] (mean)
   Time per request:       8.670 [ms] (mean)
   Time per request:       0.867 [ms] (mean, across all concurrent requests)
   Transfer rate:          756.37 [Kbytes/sec] received
   
   Connection Times (ms)
                 min  mean[+/-sd] median   max
   Connect:        0    1   0.5      1       2
   Processing:     2    7   4.2     10      12
   Waiting:        1    6   3.8      3      10
   Total:          2    8   3.9     11      12
   
   Percentage of the requests served within a certain time (ms)
     50%     11
     66%     11
     75%     12
     80%     12
     90%     12
     95%     12
     98%     12
     99%     12
    100%     12 (longest request)
   ~~~

4. 查看错误日志

   ~~~java
   2020/03/29 17:25:13 [error] 6018#6018: *67 limiting requests, excess: 0.984 by zone "req_zone", client: 192.168.x.xxx, server: localhost, request: "GET /red.html HTTP/1.0", host: "192.168.x.xxx"
   2020/03/29 17:25:13 [error] 6018#6018: *68 limiting requests, excess: 0.984 by zone "req_zone", client: 192.168.x.xxx, server: localhost, request: "GET /red.html HTTP/1.0", host: "192.168.x.xxx"
   ~~~

   说明请求限制生效。

5. 测试burst和nodelay

   打开配置：

   ~~~java
   limit_req zone=req_zone burst=3 nodelay;
   ~~~

   同样的压测条件，结果：

   ~~~java
   Server Software:        nginx/1.16.1
   Server Hostname:        192.168.1.104
   Server Port:            80
   
   Document Path:          /red.html
   Document Length:        127 bytes
   
   Concurrency Level:      10
   Time taken for tests:   0.012 seconds
   Complete requests:      20
   Failed requests:        16
      (Connect: 0, Receive: 0, Length: 16, Exceptions: 0)
   Write errors:           0
   Non-2xx responses:      16 // 减少到只有16个返回非200状态
   Total transferred:      12444 bytes
   HTML transferred:       8412 bytes
   Requests per second:    1707.80 [#/sec] (mean)
   Time per request:       5.855 [ms] (mean)
   Time per request:       0.586 [ms] (mean, across all concurrent requests)
   Transfer rate:          1037.69 [Kbytes/sec] received
   
   Connection Times (ms)
                 min  mean[+/-sd] median   max
   Connect:        0    1   0.4      1       2
   Processing:     3    3   0.8      3       5
   Waiting:        1    3   0.5      3       3
   Total:          3    4   0.8      5       5
   WARNING: The median and mean for the total time are not within a normal deviation
           These results are probably not that reliable.
   
   Percentage of the requests served within a certain time (ms)
     50%      5
     66%      5
     75%      5
     80%      5
     90%      5
     95%      5
     98%      5
     99%      5
    100%      5 (longest request)
   ~~~

   

### 四、连接限制演示

修改配置：

~~~java
limit_conn_zone $binary_remote_addr zone=conn_zone:1m;
limit_conn conn_zone 1;
~~~





