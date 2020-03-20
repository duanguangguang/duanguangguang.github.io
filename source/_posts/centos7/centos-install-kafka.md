---
title: centos单机安装kafka
date: 2020-03-20 18:57:01
categories: Centos7
tags:
  - Centos7
---

### 一、安装JDK

参考：[centos安装JDK](https://duanguangguang.github.io/2020/03/20/centos7/centos-install-jdk/ )

### 二、安装zookeeper

参考：[centos安装zookeeper](https://duanguangguang.github.io/2020/03/20/centos7/centos-install-zookeeper/ )

### 三、安装kafka

1. 解压kafka

   ~~~java
   [root@common kafka]# tar -zxvf kafka_2.13-2.4.0.tgz 
   ~~~

2. 修改kafka配置文件

   我的kafka是2.4版本的需要自己创建并配置日志路径

   ~~~java
   log.dirs=/home/duan/myhome/soft/kafka/kafka_2.13-2.4.0/logs/kafka
   ~~~

<!-- more -->

### 四、启动kafka

1. 采用自带的zookeeper

   - 启动zookeeper

     ~~~java
     nohup zookeeper-server-start.sh /home/duan/myhome/soft/kafka/kafka_2.13-2.4.0/config/zookeeper.properties &
     ~~~

   - 启动kafka

     ~~~java
     nohup kafka-server-start.sh /home/duan/myhome/soft/kafka/kafka_2.13-2.4.0/config/server.properties &
     ~~~

2. 采用自己安装的zookeeper

   - 配置zookeeper的zoo.conf

     ~~~java
     server.1=192.168.1.102:2888:3888
     ~~~

   - 创建zookeeper的myid文件

     ~~~java
     cd dataDir配置的目录下
     touch myid
     echo 1 > myid
     ~~~

   - 启动zookeeper

     ~~~java
     [root@common bin]# sh zkServer.sh start
     [root@common bin]# sh zkServer.sh status
     ZooKeeper JMX enabled by default
     Using config: /home/duan/myhome/soft/zookeeper/zookeeper3.5.7/bin/../conf/zoo.cfg
     Client port found: 2181. Client address: localhost.
     Mode: standalone
     ~~~

   - 修改kafka配置文件server.properties

     配置与zookeeper的ip相同（如果zookeeper不配置ip的话，kafka也无需再配置，采用默认localhost即可）：

     ~~~java
     zookeeper.connect=192.168.1.102:2181
     ~~~

   - 启动kafka

     ~~~java
     bin/kafka-server-start.sh config/server.properties &
     ~~~

     ![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-kafka\01kafka启动成功.png)

### 五、测试kafka

1. 创建主题

   ~~~java
   kafka-topics.sh --create --zookeeper 192.168.1.102:2181 --topic test3 --partitions 1 --replication-factor 1
   ~~~

2. 查看主题

   ~~~java
   kafka-topics.sh --list --zookeeper 192.168.1.102:2181
   ~~~

3. 创建生产者

   ~~~java
   kafka-console-producer.sh --topic test3 --broker-list localhost:9092
   ~~~

4. 创建消费者

   ~~~java
   kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test3 --from-beginning
   ~~~

5. 测试kafka

   ![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-kafka\02kafka测试成功.png)

