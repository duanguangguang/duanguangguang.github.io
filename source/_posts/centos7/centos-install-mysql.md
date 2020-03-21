---
title: centos7安装mysql5.7
date: 2020-03-21 20:23:02
categories: Centos7
tags:
  - Centos7
---

### 一、查看MySQL5.7的yum源

进入MySQL官网，或者直接点开这个地址：https://dev.mysql.com/downloads/repo/yum/ 。

![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-mysql\mysql查看yum源.png)

之所以选择7这个版本，是因为这个版本里面包含有MySQL5.5,5.6,5.7,8.0版本，最新的8版本里面只有8了。在下载页复制下载链接<https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm>。

<!-- more -->

### 二、删除老版本yum源

1. 查看MySQL安装情况

   ~~~java
   rpm -qa | grep -i mysql
   ~~~

2. 停止MySQL服务

   ~~~java
   service mysql stop
   ~~~

3. 删除已安装的rpm包

   ~~~java
   rpm -e --nodeps 文件名
   ~~~

4. 检查MySQL安装情况

   ~~~java
   rpm -qa | grep -i mysql
   ~~~

### 三、下载安装yum源

1. 下载yum源

   ~~~java
   wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
   ~~~

2. 安装命令

   继续输入安装命令：

   ~~~java
   rpm -ivh mysql57-community-release-el7-11.noarch.rpm
   ~~~

3. 修改yum默认安装版本

   若是直接下载的5.7的yum源可以不用修改此步骤。

   - 运行查看可安装的mysql的命令

     ~~~java
     yum repolist all| grep mysql
     ~~~

     ![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-mysql\mysql5.7启动.png)

   - 修改安装版本

     若上面查看要安装的版本是禁用状态，则修改安装版本：

     ~~~java
     vim /etc/yum.repos.d/mysql-community.repo
     ~~~

     ![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-mysql\修改yum版本.png)

     将5.7版本enabled设置为1，其他设置为0。

4. 通过yum安装MySQL

   ~~~java
   yum install -y mysql-community-server
   ~~~

   然后等待安装完成。 

### 四、启动MySQL服务

1. 开启数据库

   ~~~java
   systemctl start mysqld
   ~~~

2. 开启开机自启数据库

   ~~~java
   systemctl enable mysqld
   ~~~

### 五、修改MySQL的初始密码

1. 查看MySQL初始密码

   ~~~java
   grep 'password' /var/log/mysqld.log
   ~~~

   也可以自己查看：

   ~~~java
   vi /var/log/mysqld.log
   ~~~

2. 修改初始密码

   ~~~java
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'Root123@'；
   ~~~

   或者：

   ~~~java
   set password for 'root'@'localhost'=password('Root123@'); 
   ~~~

   注意：mysql5.7默认密码策略要求密码必须是大小写字母数字特殊字母的组合，至少8位。

### 六、设置远程访问

1. 设置防火墙开放3306端口

   - 打开防火墙配置文件 

     ~~~java
     vi /etc/sysconfig/iptables
     ~~~

   - 在icmp-host-prohibited前增加一行

     ~~~java
     -A INPUT -p tcp -m state --state NEW -m tcp --dport 3306 -j ACCEPT
     ~~~

   - 重启防火墙

     ~~~java
     service  iptables restart
     ~~~

   - 重启MySQL

     ~~~java
     systemctl restart mysqld
     ~~~

2. MySQL授权

   ~~~java
   GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'Root123@' WITH GRANT OPTION;
   ~~~

   这句话解决了一个问题，就是那个“%”。它允许XXX用户远程连接Mysql服务，而如果@后边的那个值是 localhost 的话，那XXX用户就只能在服务器上连接Mysql服务了。如果用户不是你建的，那就得去mysql的user表中将该用户的host改为%就行了。 

3. 本地Navicat工具连接测试

### 七、修改MySQL默认编码

1. 修改默认编码为utf-8

   修改/etc/my.cnf配置文件，在[mysqld]下添加编码配置：

   ~~~~java
   character_set_server=utf8
   init_connect='SET NAMES utf8'
   ~~~~

2. 重启MySQL

   ~~~java
    systemctl restart mysqld
   ~~~

3. 登录root用户查看编码

   ~~~java
   show variables like '%character%';
   ~~~

   ![](C:\duanguangguang.github.io\source\_posts\centos7\centos-install-mysql\mysql字符集.png)



