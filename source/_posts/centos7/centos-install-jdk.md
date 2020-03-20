---
title: centos安装jdk
date: 2020-03-20 18:56:53
categories: Centos7
tags:
  - Centos7
---

### 一、卸载openJDK

CentOS系统默认安装了openjdk的（如果操作系统不是最小安装），查看版本：`rpm -qa | grep java`或者`java -version` 。删除系统自带的OpenJDK`rpm -e --nodeps `xxx`。xxx为查询出来的jdk名字。

### 二、安装JDK

下载JDK到本机，并传输到CentOS上 。

![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-jdk\01.png)

解压JDK：

~~~java
tar -zxvf jdk-8u231-linux-x64.tar.gz
~~~

### 三、配置环境变量 

输入`vim /etc/profile`编辑环境变量，进入编辑状态。在编辑栏底部添加以下配置：

~~~java
export JAVA_HOME=/home/duan/myhome/soft/jdk/jdk1.8.0_231
export JRE_HOME=/home/duan/myhome/soft/jdk/jdk1.8.0_231/jre
export CLASSPATH=.:${JAVA_HOME}/jre/lib/rt.jar:${JAVA_HOME}/lib/dt.jar:${JAVA_HOME}/lib/tools.jar
export PATH=$PATH:${JAVA_HOME}/bin
~~~

使/etc/profile里的配置立即生效  

~~~java
source /etc/profile
~~~

### 四、测试JDK

~~~java
[root@common backup]# java -version
java version "1.8.0_231"
Java(TM) SE Runtime Environment (build 1.8.0_231-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.231-b11, mixed mode)
~~~





