---
title: nginx https服务
date: 2020-04-04 11:47:24
categories: nginx
tags:
  - nginx
---

### 介绍

因为HTTP协议是不安全的，会发生下面几点情况：

- 传输数据被中间人盗用、信息泄露
- 数据内容劫持、篡改

所以需要使用HTTPS协议，有了HTTPS就能解决HTTP传输不安全的问题，因为HTTPS对传输内容进行加密以及身份验证。

<!-- more -->

### 一、加密方式

1. 对称加密

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-https\对称加密.png)

   明文的数据在发送方进行数据的加密，在接收方进行数据的解密，发送方用的加密秘钥，接收方用的解密秘钥，对称加密加密秘钥和解密秘钥是一对的，一样的秘钥。

2. 非对称加密

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-https\非对称加密.png)

   非对称加密的加密秘钥和解密秘钥是两串秘钥，是不一样的，分为公钥和私钥，只有通过公钥加密的东西，才能通过对应的私钥进行解密。

3. HTTPS加密协议原理

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-https\https加密协议原理.png)

   同时用到了加密和非对称加密，首先当用户端发起SSL连接的时候，进行的是非对称加密，服务端保管唯一私钥，将公钥发送给客户端，客户端收到公钥之后，再用公钥加密接下来要进行对称加密的密码，并发送到服务端。

   HTTPS采用对称加密和非对称加密的原因：因为非对称对连接要求较高，多次的连接情况下对性能会有所损耗，对称加密对性能来说比较简单，所以在第一次连接之后，完全可以用对称加密的方式了。

4. 中间人劫持

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-https\中间人劫持.png)

   中间人可以在前面进行握手的时候，对客户端发送服务端的数据进行劫持，伪装成客户端，并且在服务端发送数据给客户端时也能伪装成服务端，进行劫持，对多次连接进行劫持。解决这个问题就用到了HTTPS的CA证书。

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-https\CA证书.png)

   服务端想客户端之前是发送的公钥，现在发送的是CA签名证书，这时中间人劫持就没有用了，中间人无法生成一个匹配CA签名证书的校验的，因为客户端会对CA证书进行校验，与第三方机构进行校验，如果校验成功则利用公钥加密，CA证书包含对应的公钥，如果校验失败，则停止会话。

### 二、HTTPS生成CA证书

介绍自签证书。对于生成秘钥和CA证书，首先确认系统有没有装openssl。

1. 查看系统openssl

   ~~~java
   [root@node1 conf.d]# openssl version
   OpenSSL 1.0.2k-fips  26 Jan 2017
   ~~~

2. 查看nginx有没有编译http_ssl_module

   ~~~java
   [root@node1 conf.d]# nginx -V
   nginx version: nginx/1.16.1
   built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
   built with OpenSSL 1.0.2k-fips  26 Jan 2017
   TLS SNI support enabled
   configure arguments: --with-http_ssl_module
   ~~~

生成证书步骤：

1. 生成key秘钥

   ~~~java
   [root@node1 ssl_key]# openssl genrsa -idea -out duan.key 1024
   Generating RSA private key, 1024 bit long modulus
   ........++++++
   ...................................................++++++
   e is 65537 (0x10001)
   Enter pass phrase for duan.key:
   Verifying - Enter pass phrase for duan.key:
   [root@node1 ssl_key]# ls
   duan.key
   ~~~

   - idea：这时一种对称加密的算法
   - out：key文件
   - 1024：位数越高，精度越高

2. 生成证书签名请求文件（csr文件）

   ~~~java
   [root@node1 ssl_key]# openssl req -new -key duan.key -out duan.csr
   Enter pass phrase for duan.key:
   You are about to be asked to enter information that will be incorporated
   into your certificate request.
   What you are about to enter is what is called a Distinguished Name or a DN.
   There are quite a few fields but you can leave some blank
   For some fields there will be a default value,
   If you enter '.', the field will be left blank.
   -----
   Country Name (2 letter code) [XX]:CN
   State or Province Name (full name) []:shenzhen
   Locality Name (eg, city) [Default City]:shenzhen
   Organization Name (eg, company) [Default Company Ltd]:CN
   Organizational Unit Name (eg, section) []:dodd
   Common Name (eg, your name or your server's hostname) []:node1
   Email Address []:1@qq.com
   
   Please enter the following 'extra' attributes
   to be sent with your certificate request
   A challenge password []:
   An optional company name []:dodd
   [root@node1 ssl_key]# ls
   duan.csr  duan.key
   ~~~

   - new：表示生成一个新文件
   - 有了这两个文件，就可以找第三方机构进行CA证书签名

3. 生成证书签名文件（CA文件）

   ~~~java
   [root@node1 ssl_key]# openssl x509 -req -days 3650 -in duan.csr -signkey duan.key -out duan.crt
   Signature ok
   subject=/C=CN/ST=shenzhen/L=shenzhen/O=CN/OU=dodd/CN=node1/emailAddress=1@qq.com
   Getting Private key
   Enter pass phrase for duan.key:
   [root@node1 ssl_key]# ls
   duan.crt  duan.csr  duan.key
   ~~~

   - days 3650：表示签名证书的过期时间，默认一个月
   - in：表示需要将csr文件加入进去

nginxdeHTTPS语法配置：

~~~java
Syntax:	ssl on|off;
Default: ssl off;
Context: http,server

// 证书文件
Syntax:	ssl_certificate file;
Default: --
Context: http,server

// 证书密码文件
Syntax:	ssl_certificate_key file;
Default: --
Context: http,server
~~~

场景演示：

1. 演示html文件

   ~~~java
   <html>
   <head>
   	<meta charset="utf-8">
   	<title>admin</title>
   </head>
   <body style="background-color:red;">
   <h1>Admin</h1>
   </body>
   </html>
   ~~~

2. 配置nginx

   ~~~java
   server
    {
      listen       443;
      server_name  192.168.x.xxx node1;
      ssl on;
      ssl_certificate /etc/nginx/ssl_key/duan.crt;
      ssl_certificate_key /etc/nginx/ssl_key/duan.key;
      #ssl_certificate_key /etc/nginx/ssl_key/duan_nopass.key;
   
      index index.html index.htm;
      location / {
          root  /opt/app/code;
      }
   }
   ~~~

3. 重启nginx服务

   ~~~java
   [root@node1 conf.d]# nginx -s stop -c /etc/nginx/nginx.conf 
   nginx: [warn] the "ssl" directive is deprecated, use the "listen ... ssl" directive instead in /etc/nginx/conf.d/test_https.conf:5
   Enter PEM pass phrase:
   [root@node1 conf.d]# nginx -c /etc/nginx/nginx.conf 
   nginx: [warn] the "ssl" directive is deprecated, use the "listen ... ssl" directive instead in /etc/nginx/conf.d/test_https.conf:5
   Enter PEM pass phrase:
   ~~~

   发现每次重启服务都需要输入密码。

4. 查看系统有没有开启443端口监听

   ~~~java
   [root@node1 conf.d]# netstat -luntp|grep 443
   tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      4988/nginx: master  
   ~~~

5. 访问测试

   ~~~java
   https://192.168.x.xxx/admin.html
   ~~~

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-https\ca证书演示一.png)

   继续访问：

   ![](C:\duanguangguang.github.io\source\_posts\nginx\nginx-https\ca证书演示二.png)

### 三、配置苹果要求的证书

要求：

- 服务器所有的连接使用TLS1.2以上版本（openssl1.0.2）
- HTTPS证书必须使用SHA256以上哈希算法签名
- HTTPS证书必须使用RSA 2048位或ECC 256位以上公钥算法
- 使用前向加密技术

查看openssl版本：

~~~java
[root@node1 ssl_key]# openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017
~~~

可以看到版本是1.0.2。

查看加密算法：

~~~java
[root@node1 ssl_key]# openssl x509 -noout -text -in ./duan.crt
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            f2:d8:fe:44:4a:95:54:66
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=shenzhen, L=shenzhen, O=CN, OU=dodd, CN=node1/emailAddress=1@qq.com
        Validity
            Not Before: Apr  5 09:06:28 2020 GMT
            Not After : Apr  3 09:06:28 2030 GMT
        Subject: C=CN, ST=shenzhen, L=shenzhen, O=CN, OU=dodd, CN=node1/emailAddress=1@qq.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (1024 bit)
~~~

可以看到加密算法使用：sha256，位数：1024。

直接通过key生成crt文件：

~~~java
openssl req -days 36500 -x509 -sha256 -nodes -newkey rsa:2048 -keyout duan.key -out duan_apple.crt
~~~

替换证书，修改nginx配置文件：

~~~java
#ssl_certificate /etc/nginx/ssl_key/duan.crt;
ssl_certificate /etc/nginx/ssl_key/duan_apple.crt;
~~~

去掉保护码：

~~~java
openssl rsa -in ./duan.key -out ./duan_nopass.key
~~~

### 四、HTTPS服务优化

HTTPS的建立是要在HTTP之前建立ssl握手，HTTPS的认证就会多一次连接，对于服务端就会需要进行对应的认证，会消耗服务端的cpu资源以及io资源。

方式一：激活keepalive长连接

方式二：设置ssl session缓存

~~~java
server
 {
   listen       443;
   server_name  192.168.x.xxx node1;
 
   keepalive_timeout 100;

   ssl on;
   ssl_session_cache   shared:SSL:10m;
   ssl_session_timeout 10m;

   #ssl_certificate /etc/nginx/ssl_key/duan.crt;
   ssl_certificate /etc/nginx/ssl_key/duan_apple.crt;
   ssl_certificate_key /etc/nginx/ssl_key/duan.key;
   #ssl_certificate_key /etc/nginx/ssl_key/duan_nopass.key;

   index index.html index.htm;
   location / {
       root  /opt/app/code;
   }
}
~~~





