---
title: nginx作为负载均衡服务
date: 2020-03-31 23:23:47
categories: nginx
tags:
  - nginx
---

### 介绍

对于请求而言，负载均衡能很好的均摊请求，提高服务端吞吐率和整体性能，多个服务节点部署的方式，也提高了容灾和服务高可用。负载均衡分为:

1. GSLB

   全局负载均衡，往往按照国家为单位或省为单位来进件负载均衡的。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-load-balance\gslb.png)

   北京用户张三，他不可能直接就去请求调度中心节点，先请求北京地区的调度节点，调度节点返回给张三对应的地址，张三再访问应用服务，这个应用服务也在北京。所以这样没有给中心节点造成压力。

2. SLB

   往往调度节点和服务节点在一个逻辑单元里面，或者说是在一个地域里面，那么小的地域就决定了他对部分服务的实时性，响应性是非常好的。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-load-balance\slb.png)

   nginx就是一个典型的SLB。

<!-- more -->

### 一、负载均衡分类

负载均衡按照网络模型（OSI）可以分为：

1. 四层负载均衡

   四层负载均衡就是在OSI模型中的传输层，传输层可以支持到tcp/ip协议的控制，所以他只需要对客户端的请求进行tcp/ip协议的包转发，就可以实现负载均衡。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-load-balance\4层负载均衡.png)

   好处是性能非常快，只需要最底层进行应用处理，二不需要进行复杂的逻辑，只需要进行包的转发就可以。

2. 七层负载均衡

   七层负载均衡是在应用层，所有可以完成很多应用方面的协议的请求，比如：http应用层负载均衡，他可以实现http信息的改写，头信息的改写，安全应用规则的控制，以及转发，rewrite等规则。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-load-balance\7层负载均衡.png)

   nginx就是一个典型的7层SLB。

### 二、配置语法

nginx实现负载均衡的原理：就是用到了`proxy_pass`，它把所有客户端的请求代理去转发到对应的后端服务器上，只是它不是转发到一台服务，而是转发到一组虚拟的服务池中。

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-load-balance\负载均衡原理.png)

配置语法：

~~~java
Syntax:	upstream name {...};
Default: --
Context: http
~~~

配置场景：

~~~java
upstream load {
        server 192.168.x.xx1:8001;
        server 192.168.x.xx2:8002;
        server 192.168.x.xx3:8003;
    }
server { 
    listen       80;
    server_name  localhost node1;
    
    location / {
        proxy_pass http://load;
        include proxy_params;
    }
}
~~~

使用iptables规则关闭8002服务：

~~~java
iptables -I INPUT -p tcp --dport 8002 -j DROP
~~~

访问测试：

~~~java
http://node1
~~~

### 三、upstream参数介绍

~~~java
upstream backend {
    ip_hash;
    server backend1.example.com weight=5;
    server backend2.example.com:8080;
    server unix:/tmp/backend3;
    
    server backend1.example.com:8080 down;
    server backend2.example.com:8080 backup;
    server backend3.example.com:8080 max_fails=1 fail_timeout=10s;
}
~~~

后端服务器在负载均衡调度中的状态：

| 状态         | 描述                                         |
| ------------ | -------------------------------------------- |
| down         | 当前的server暂时不参与负载均衡               |
| backup       | 预留的备份服务器，其他节点都不提供服务时使用 |
| max_fails    | 允许请求失败的次数                           |
| fail_timeout | 经过max_fails失败后，服务暂停的时间          |
| max_conns    | 限制最大的接收的连接数                       |

### 四、调度算法

| 调度算法     | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| 轮询         | 按照时间 顺序逐一分配到不同的后端服务器（默认）              |
| 加权轮询     | weight值越大，分配到的访问几率越高                           |
| ip_hash      | 每个请求按访问IP的hash结果分配，这样来自同一个IP的固定访问一个后端服务器 |
| url_hash     | 按照访问的URL的hash结果来分配请求，使每个URL定向到同一个后端服务器 |
| least_conn   | 最少连接数，那个机器连接数少就分发                           |
| hash关键数值 | hash自定义的key                                              |

url_hash配置：

~~~java
upstream backend {
    hash $request_url;
    server backend1.example.com 8081;
    server backend2.example.com 8082;
    server backend3.example.com 8083;
}
~~~

hash key配置：

~~~java
Syntax:	hash key [xonsistent];
Default: --
Context: upstream
This directive appeared in version 1.7.2
~~~









