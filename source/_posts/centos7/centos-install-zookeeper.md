---
title: centos单机安装zookeeper
date: 2020-03-20 18:18:23
categories: Centos7
tags:
  - Centos7
---

### 一、下载安装包

从版本3.5.5开始，带有bin名称的包才是我们想要的下载可以直接使用的里面有编译后的二进制的包，而之前的普通的tar.gz的包里面是只是源码的包无法直接使用。 zookeeper安装需要jdk环境，参考：[centos安装JDK](https://duanguangguang.github.io/2020/03/20/centos7/centos-install-jdk/ )

### 二、下载以及解压 

下载地址：http://mirror.bit.edu.cn/apache/zookeeper/ 

解压：

~~~java
[root@common backup]# tar -zxvf apache-zookeeper-3.5.7-bin.tar.gz 
~~~

目录结构：

![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-zookeeper\01zookeeper安装目录结构.png)

<!-- more -->

- bin目录：zk的可执行脚本目录，包括zk服务进程，zk客户端，等脚本。其中.sh是Linux环境下的脚本，.cmd是Windows环境下的脚本
- conf目录：配置文件目录。zoo_sample.cfg为样例配置文件，需要修改为自己的名称，一般为zoo.cfg。log4j.properties为日志配置文件
- lib目录：zk依赖的包
- docs目录：zk某些用法的代码示例

### 三、修改配置文件

复制一个zoo_sample.cfg文件为zoo.cfg： 

~~~java
cp zoo_sample.cfg zoo.cfg
~~~

单机模式无需任何配置修改，集群模式需要部署zk进程。zoo.cfg文件说明：

~~~java
[root@common conf]# cat zoo.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
maxClientCnxns=60
clientPort=2181
#server.1=192.168.1.102:2888:3888
~~~

- initLimit：ZooKeeper集群模式下包含多个zk进程，其中一个进程为leader，余下的进程为follower。当follower最初与leader建立连接时，它们之间会传输相当多的数据，尤其是follower的数据落后leader很多。initLimit配置follower与leader之间建立连接后进行同步的最长时间

- syncLimit：配置follower和leader之间发送消息，请求和应答的最大时间长度

- tickTime：tickTime则是上述两个超时配置的基本单位，例如对于initLimit，其配置值为5，说明其超时时间为 2000ms * 5 = 10秒

- server.id=host:port1:port2：其中id为一个数字，表示zk进程的id，这个id也是dataDir目录下myid文件的内容。host是该zk进程所在的IP地址，port1表示follower和leader交换消息所使用的端口，port2表示选举leader所使用的端口

- dataDir：其配置的含义跟单机模式下的含义类似，不同的是集群模式下还有一个myid文件。myid文件的内容只有一行，且内容只能为1 - 255之间的数字，这个数字亦即上面介绍server.id中的id，表示zk进程的id

  ~~~java
  touch myid
  echo 1 > myid
  ~~~

### 四、启动zookeeper

1. 进入bin目录

   ~~~java
   [root@common zookeeper3.5.7]# cd bin
   ~~~

2. ./zkServer.sh start  以后台方式启动服务器

   ~~~java
   [root@common bin]# ./zkServer.sh start
   ZooKeeper JMX enabled by default
   Using config: /home/duan/myhome/soft/zookeeper/zookeeper3.5.7/bin/../conf/zoo.cfg
   Starting zookeeper ... STARTED
   ~~~

3. 如果想在前台中运行以便查看服务器进程的输出日志，可以通过以下命令运行  ./zkServer.sh start-foreground

   ~~~java
   [root@common bin]# ./zkServer.sh start-foreground
   ZooKeeper JMX enabled by default
   Using config: /home/duan/myhome/soft/zookeeper/zookeeper3.5.7/bin/../conf/zoo.cfg
   2020-03-20 18:44:02,519 [myid:] - INFO  [main:QuorumPeerConfig@135] - Reading configuration from: /home/duan/myhome/soft/zookeeper/zookeeper3.5.7/bin/../conf/zoo.cfg
   2020-03-20 18:44:02,548 [myid:] - INFO  [main:QuorumPeerConfig@387] - clientPortAddress is 0.0.0.0:2181
   ~~~

4. ./zkCli.sh  启动客户端 

   ~~~java
   [root@common bin]# ./zkCli.sh
   ~~~

   ![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-zookeeper\02zkCli启动成功.png)

5. 查看zookeeper状态

   - jps方式

     ~~~java
     [root@common bin]# jps
     11693 Jps
     11535 QuorumPeerMain ---有该节点启动则成功
     ~~~

   - status方式

     ~~~java
     [root@common bin]# ./zkServer.sh status
     ZooKeeper JMX enabled by default
     Using config: /home/duan/myhome/soft/zookeeper/zookeeper3.5.7/bin/../conf/zoo.cfg
     Client port found: 2181. Client address: localhost.
     Mode: standalone
     ~~~

6. ./zkServer.sh stop 停止服务器

   ~~~java
   [root@common bin]# ./zkServer.sh stop
   ZooKeeper JMX enabled by default
   Using config: /home/duan/myhome/soft/zookeeper/zookeeper3.5.7/bin/../conf/zoo.cfg
   Stopping zookeeper ... STOPPED
   ~~~

### 六、配置环境变量

1. 想要全局启动服务器或者客户端需要配置全局变量，vim /etc/profile在文件末尾追加：

   ~~~java
   export ZK_HOME=/home/duan/myhome/soft/zookeeper/zookeeper3.5.7
   export PATH=$PATH:$ZK_HOME/bin
   ~~~

2. 使环境变量立即生效

   ~~~java
   source /etc/profile
   ~~~

   





