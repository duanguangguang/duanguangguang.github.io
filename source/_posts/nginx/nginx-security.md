---
title: nginx安全
date: 2020-04-05 23:50:05
categories: nginx
tags:
  - nginx
---

### 介绍

常见的恶意行为：爬虫行为和恶意抓取、资源盗用

- 基础防盗链功能，目的不让恶意用户能轻易的爬取网站对外数据
- secure_link_module对数据安全性提高加密验证和失效性，适合核心重要数据
- access_module对后台、部分用户服务的数据提供IP防控

<!-- more -->

### 一、常见的攻击手段

1. 后台密码撞库

   通过猜测密码字典不断对后台系统登录性尝试，获取后台登录密码。

   - 方法一：后台登录密码复杂度
   - 方法二：access_module对后台提供IP防控
   - 方法三：预警机制，结合nginx和lua开发

2. 文件上传漏洞

   利用这些可以上传的接口将恶意代码植入到服务器中，再通过url去访问可以执行代码。例：

   ~~~java
   http://www.imooc.com/upload/1.jpg/1.php
   Nginx将1.jpg作为PHP代码执行
   ~~~

   nginx防护配置：

   ~~~java
   location ^~/upload {
       root /opt/app/images;
       if ($request_filename ~*(.*)\.php) {
           return 403;
       }
   }
   ~~~

3. SQL注入

   利用未过滤、未审核用户输入的攻击方法，让应用运行本不应该运行的SQL代码。

### 二、SQL注入场景演示

SQL注入场景演示：

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\sql注入场景演示.png)

准备mariadb和lnmp环境：

~~~java
[root@node1 work]# yum install mariadb-server mariadb
[root@node1 work]# yum install php php-fpm php-mysql
~~~

建立表格，插入测试数据：

~~~java
// 启动mysql
[root@node1 /]# systemctl start mysqld
[root@node1 /]# ps -ef|grep mysql
mysql      1848      1  0 14:33 ?        00:00:10 /usr/sbin/mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
root       7632   2224  0 18:03 pts/0    00:00:00 grep --color=auto mysql
[root@node1 /]# mysql -uroot -p

// 插入测试数据
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| xxl_job            |
+--------------------+
5 rows in set (0.01 sec)

mysql> create database info;
Query OK, 1 row affected (0.00 sec)

mysql> use info;
Database changed
mysql> show tables;
Empty set (0.00 sec)

mysql> create table users(id int(11),username varchar(64),password varchar(64),email varchar(64));
Query OK, 0 rows affected (0.02 sec)

mysql> insert into users (id,username,password,email) values(1,'duan','123','1@qq.com');
Query OK, 1 row affected (0.01 sec)
    
mysql> insert into users (id,username,password,email) values(2,'duan',md5('123'),'1@qq.com');
Query OK, 1 row affected (0.01 sec)
    
mysql> select * from users;
+------+----------+----------------------------------+----------+
| id   | username | password                         | email    |
+------+----------+----------------------------------+----------+
|    1 | duan     | 123                              | 1@qq.com |
+------+----------+----------------------------------+----------+
|    2 | duan     | 202cb962ac59075b964b07152d234b70 | 1@qq.com |
+------+----------+----------------------------------+----------+
2 row in set (0.00 sec)
~~~

修改nginx配置：

~~~java
location ~ \.php$ {
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        #fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        fastcgi_param  SCRIPT_FILENAME   /opt/app/code/$fastcgi_script_name;
        include        fastcgi_params;
    }
~~~

启动php-fpm：

~~~java
[root@node1 conf.d]# php-fpm -D
[root@node1 conf.d]# ps -aux|grep php
root       7782  0.0  0.2 256560  4844 ?        Ss   18:17   0:00 php-fpm: master process (/etc/php-fpm.conf)
apache     7783  0.0  0.2 258644  5044 ?        S    18:17   0:00 php-fpm: pool www
apache     7784  0.0  0.2 258644  5044 ?        S    18:17   0:00 php-fpm: pool www
apache     7785  0.0  0.2 258644  5044 ?        S    18:17   0:00 php-fpm: pool www
apache     7786  0.0  0.2 258644  5044 ?        S    18:17   0:00 php-fpm: pool www
apache     7787  0.0  0.2 258644  5044 ?        S    18:17   0:00 php-fpm: pool www
root       7789  0.0  0.0 110296   900 pts/0    R+   18:17   0:00 grep --color=auto php
~~~

确定lnmp环境可用：在/opt/app/code目录下新建phpinfo.php 

~~~java
[root@node1 code]# cat phpinfo.php 
<?php
    phpinfo();
?>
~~~

重载nginx服务：

~~~java
[root@node1 conf.d]# nginx -s reload -c /etc/nginx/nginx.conf
~~~

访问测试：

~~~java
http://192.168.x.xxx/phpinfo.php
~~~

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\phpinfo.png)

至此，lnmp环境配置成功。

场景演示：

1. 演示文件

   login.html

   ~~~java
   <html>
   <head>
   <title>Jeson Sql注入演示</title>
   <meta http-equiv="content-type"content="text/html;charset=utf-8">
   </head>
   <body>
   <form action="validate.php" method="post">
       <table>
         <tr>
           <td>用户名：</td>
           <td><input type="text" name="username"></td>
         </tr>
         <tr>
           <td>密  码：</td>
           <td><input type="text" name="password"></td>
         </tr>
         <tr>
           <td><input type="submit" value="提交"></td>
           <td><input type="reset" value="重置"></td>
         </tr>
       </table>
   </form>
   </body>
   </html>
   ~~~

   validate.php

   ~~~java
   <?php
         $conn = mysql_connect("localhost",'root','') or die("数据库连接失败！");
         mysql_select_db("info",$conn) or die("您要选择的数据库不存在");
         $name=$_POST['username'];
         $pwd=$_POST['password'];
         $sql="select * from users where username='$name' and password='$pwd'";
         echo $sql."<br /><br />";
         $query=mysql_query($sql);
         $arr=mysql_fetch_array($query);
         if($arr){
             echo "login success!\n";
             echo $arr[1];
             echo $arr[3]."<br /><br />";
         }else{
             echo "login failed!";
         }
       
         #if(is_array($arr)){
         #       header("Location:manager.php");
         #}else{
         #       echo "您的用户名或密码输入有误，<a href="Login.php">请重新登录！</a>";
         #}
   ?>
   ~~~

2. 重载nginx服务后访问测试：

   ~~~java
   http://192.168.x.xxx/login.html
   ~~~

   输入错误用户名密码：

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\错误用户.png)

   输入正确用户名密码：

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\正确用户.png)

   

3. SQL注入演示

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\sql注入登录信息.png)

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\sql注入成功.png)

   请求sql相当于：`select * from users where username = '' or 1 = 1`。

### 三、Nginx+Lua防火墙

Nginx+Lua来进行所有用户信息审核和检查，原理是对关键字的检测。防火墙功能：

![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\waf防火墙功能.png)

github地址：[Nginx+Lua防火墙](https://github.com/loveshell/ngx_lua_waf)

1. 代码下载

   ~~~java
   [root@node1 opt]# cd /opt/download
   [root@node1 opt]# yum install git
   [root@node1 download]# git clone https://github.com/loveshell/ngx_lua_waf
   [root@node1 download]# cd ngx_lua_waf/
   [root@node1 ngx_lua_waf]# ls
   config.lua  init.lua  install.sh  README.md  wafconf  waf.lua
   ~~~

2. 将ngx_lua_waf移动到nginx目录下waf目录下：

   ~~~java
   [root@node1 nginx]# mkdir waf
   [root@node1 nginx]# mv /opt/download/ngx_lua_waf/ ./waf/
   [root@node1 waf]# ls
   config.lua  init.lua  install.sh  README.md  wafconf  waf.lua
   ~~~

3. 更改config.lua，修改为自己对应的路径

   ~~~java
   RulePath = "/etc/nginx/waf/wafconf/"
   attacklog = "on"
   logdir = "/var/log/nginx/log/hack/"
   ~~~

4. wafconf下有对应的所有关键字规则，例post：添加or规则

   ~~~java
   [root@node1 wafconf]# cat post
   \sor\s+
   select.+(from|limit)
   (?:(union(.*?)select))
   having|rongjitest
   sleep\((\s*)(\d*)(\s*)\)
   benchmark\((.*)\,(.*)\)
   base64_decode\(
   (?:from\W+information_schema\W)
   (?:(?:current_)user|database|schema|connection_id)\s*\(
   (?:etc\/\W*passwd)
   into(\s+)+(?:dump|out)file\s*
   group\s+by.+\(
   xwork.MethodAccessor
   (?:define|eval|file_get_contents|include|require|require_once|shell_exec|phpinfo|system|passthru|preg_\w+|execute|echo|print|print_r|var_dump|(fp)open|alert|showmodaldialog)\(
   xwork\.MethodAccessor
   (gopher|doc|php|glob|file|phar|zlib|ftp|ldap|dict|ogg|data)\:\/
   java\.lang
   \$_(GET|post|cookie|files|session|env|phplib|GLOBALS|SERVER)\[
   \<(iframe|script|body|img|layer|div|meta|style|base|object|input)
   (onmouseover|onerror|onload)\=
   ~~~

   - \sor\s+：空格，中间是or，又是一个或多个空格

5. nginx集成waf，在nginx.conf下添加下面几行即可

   ~~~java
   lua_package_path "/etc/nginx/waf/?.lua";
   lua_shared_dict limit 10m;
   init_by_lua_file  /etc/nginx/waf/init.lua;
   access_by_lua_file /etc/nginx/waf/waf.lua;
   ~~~

6. 重载nginx服务

   ~~~java
   [root@node1 wafconf]# nginx -s reload -c /etc/nginx/nginx.conf
   ~~~

   

7. 访问测试

   ~~~java
   http://192.168.x.xxx/validate.php
   ~~~

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\waf防火墙1.png)

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-security\waf防火墙2.png)

### 四、CC攻击方式

对单个IP进行频繁的访问限制，可以使用到CCDeny：修改config.lua：

~~~java
CCDeny="on"
CCrate="600/60" //每60秒访问600个
~~~

正常访问测试：

~~~java
[root@node1 waf]# curl -I http://192.168.x.xxx/admin.html
HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Mon, 06 Apr 2020 14:39:05 GMT
Content-Type: text/html
Content-Length: 143
Last-Modified: Mon, 30 Mar 2020 14:56:37 GMT
Connection: keep-alive
ETag: "5e8208a5-8f"
Accept-Ranges: bytes
~~~

非正常的访问测试：

~~~java
[root@node1 waf]# ab -n 2000 -c 200 http://192.168.x.xxx/admin.html

[root@node1 waf]# curl -I http://192.168.x.xxx/admin.html
HTTP/1.1 503 Service Temporarily Unavailable
Server: nginx/1.16.1
Date: Mon, 06 Apr 2020 14:40:28 GMT
Content-Type: text/html
Content-Length: 197
Connection: keep-alive
~~~

### 五、Nginx自身漏洞

1. nginx1.20版本自身漏洞

   CVE-2017-7529。这个漏洞可以利用分片请求这个模块通过客户端的一些方式取文件里面一些IP的信息，造成信息泄露。

2. 查看版本更新描述

   [nginx1.16.1更新描述](http://nginx.org/en/CHANGES-1.16 )



