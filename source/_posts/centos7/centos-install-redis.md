---
title: centos7安装redis
date: 2020-03-20 17:38:28
categories: Centos7
tags:
  - Centos7
---

### 一、安装gcc依赖

由于 redis 是用 C 语言开发，安装之前必先确认是否安装 gcc 环境（gcc -v），如果没有安装，执行以下命令进行安装 。切换到root用户下，执行yum -y install gcc命令： 

~~~java
[root@common soft]# yum install -y gcc
~~~

### 二、下载并解压安装包

~~~java
[root@common redis]# wget http://download.redis.io/releases/redis-5.0.3.tar.gz
[root@common redis]# tar -zxvf redis-5.0.3.tar.gz
~~~

<!-- more -->

### 三、执行编译

cd切换到redis解压目录下，执行编译 :

~~~java
[root@common redis]# cd redis-5.0.3
[root@common redis-5.0.3]# make
~~~

### 四、安装

安装并指定安装目录 :

~~~java
[root@common redis-5.0.3]# make install PREFIX=/home/duan/myhome/soft/redis
~~~

### 五、直接启动

1. src下启动，切换到解压的src目录下：

   ~~~java
   [root@common src]# ./redis-server
   ~~~

2. 安装目录下启动，切换到安装的bin目录下：

   ~~~java
   [root@common bin]# ./redis-server
   ~~~

3. 启动成功

   ![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-redis\01-redis启动成功.png)

   如上图：redis启动成功，但是这种启动方式需要一直打开窗口，不能进行其他操作，不太方便。

   按 ctrl + c可以关闭窗口。

### 六、后台启动

以后台进程方式启动redis。

1. 修改redis.conf文件

   将

   ~~~java
   daemonize no
   ~~~

   修改为

   ~~~java
   daemonize yes
   ~~~

2. 指定redis.conf文件启动

   ~~~java
   [root@common src]# ./redis-server /home/duan/myhome/soft/redis/redis5.0.3/redis.conf
   11077:C 20 Mar 2020 18:07:32.227 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
   11077:C 20 Mar 2020 18:07:32.227 # Redis version=5.0.3, bits=64, commit=00000000, modified=0, pid=11077, just started
   11077:C 20 Mar 2020 18:07:32.227 # Configuration loaded
   ~~~

3. 关闭redis进程

   首先使用ps -aux | grep redis查看redis进程

   ~~~java
   [root@common src]# ps -aux | grep redis
   ~~~

   使用kill命令杀死进程

   ~~~java
   [root@common src]# ps -aux | grep redis
   root  11078  0.2  0.4 153888  7704 ?  Ssl  18:07   0:00 ./redis-server 127.0.0.1:6379
   root  11091  0.0  0.0 112728   972 pts/1  R+   18:09   0:00 grep --color=auto redis
   [root@common src]# kill 11078
   ~~~

### 七、测试redis

1. 启动redis客户端

   需要修改密码，修改redis.conf文件

   ~~~java
   requirepass 123456  ----注释取消掉设置账号密码
   ~~~

   重新启动redis服务

   ~~~java
   [root@common src]# ./redis-server /home/duan/myhome/soft/redis/redis5.0.3/redis.conf
   ~~~

   启动redis客户端

   ~~~java
   [root@common bin]# ./redis-cli -h 127.0.0.1 -p 6379 -a "123456"
   ~~~

2. 测试redis

   使用ping命令，查看客户端是否连接到Redis服务器
   ~~~java
   127.0.0.1:6379> ping
   PONG
   127.0.0.1:6379> info
   # Server
   redis_version:5.0.3
   redis_git_sha1:00000000
   redis_git_dirty:0
   ~~~

   出现PONG，说明连接成功。使用info命令 ，查看相关信息，如redis服务器，redis客户端信息等。

### 八、停止redis

redis-cli shutdown  或者 kill redis进程的pid 

~~~java
[root@common bin]# ./redis-cli shutdown
[root@common bin]# ps -ef | grep redis
root      11193   5400  0 18:17 pts/0    00:00:00 grep --color=auto redis
[root@common bin]# 
~~~



