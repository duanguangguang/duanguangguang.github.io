---
title: JAVA基础(十二)java操作数据库
date: 2017-08-22 17:15:09
categories: 
 - JAVA
 - JavaSE
tags:
 - JavaSE
---

## java基础之连接数据库

java访问数据库主要用的方法是JDBC，它是java语言中用来规范客户端程序如何来访问数据库的应用程序接口，提供了诸如查询和更新数据库中数据的方法

<!-- more -->

### 一、java访问数据库的步骤

#### 1. 加载数据库

驱动加载就是把各个数据库提供的访问数据库的API加载到我们程序进来，加载JDBC驱动，并将其注册到DriverManager中，每一种数据库提供的数据库驱动不一样，加载驱动时要把jar包添加到lib文件夹下，一些主流数据库的JDBC驱动加裁注册的代码:

~~~java
//Oracle8/8i/9iO数据库(thin模式)
Class.forName("oracle.jdbc.driver.OracleDriver").newInstance();

//Sql Server7.0/2000数据库
Class.forName("com.microsoft.jdbc.sqlserver.SQLServerDriver");

//Sql Server2005/2008数据库
Class.forName("com.microsoft.sqlserver.jdbc.SQLServerDriver");

//DB2数据库
Class.froName("com.ibm.db2.jdbc.app.DB2Driver").newInstance();

//MySQL数据库 
Class.forName("com.mysql.jdbc.Driver").newInstance();
~~~

#### 2. 建立连接

建立数据库之间的连接是访问数据库的必要条件，建立连接对于不同数据库也是不一样的，一些主流数据库建立数据库连接，取得Connection对象的不同方式：

~~~java
//Oracle8/8i/9i数据库(thin模式)
String url="jdbc:oracle:thin:@localhost:1521:orcl";
String user="scott";
String password="tiger";
Connection conn=DriverManager.getConnection(url,user,password);

//Sql Server7.0/2000/2005/2008数据库
String url="jdbc:microsoft:sqlserver://localhost:1433;DatabaseName=pubs";
String user="sa";
String password="";
Connection conn=DriverManager.getConnection(url,user,password);

//DB2数据库
String url="jdbc:db2://localhost:5000/sample";
String user="amdin"
String password=-"";
Connection conn=DriverManager.getConnection(url,user,password);

//MySQL数据库
String url="jdbc:mysql://localhost:3306/testDB?user=root&password=root&useUnicode=true&characterEncoding=gb2312";
Connection conn=DriverManager.getConnection(url);
~~~

#### 3. 执行SQL语句

数据库连接建立好之后，接下来就是一些准备工作和执行sql语句了，准备工作要做的就是建立Statement对象PreparedStatement对象，例如：

~~~java
//建立Statement对象
Statement stmt=conn.createStatement();

//建立PreparedStatement对象
String sql="select * from user where userName=? and password=?";
PreparedStatement pstmt=Conn.prepareStatement(sql);
pstmt.setString(1,"admin");
pstmt.setString(2,"liubin");

//做好准备工作之后就可以执行sql语句了，执行sql语句：
String sql="select * from users";
ResultSet rs=stmt.executeQuery(sql);

//执行动态SQL查询
ResultSet rs=pstmt.executeQuery();

//执行insert update delete等语句，先定义sql
stmt.executeUpdate(sql);
~~~

#### 4. 处理结果集

访问结果记录集ResultSet对象。例如：

~~~java
while(rs.next){
//...
}
~~~

#### 5. 关闭数据库

依次将ResultSet、Statement、PreparedStatement、Connection对象关 闭，释放所占用的资源。例如：

~~~java
rs.close();
stmt.clost();
pstmt.close();
con.close();
~~~

### 二、JDBC事务

事务，就是一组操作数据库的动作集合。事务是现代数据库理论中的核心概念之一。如果一组处理步骤或者全部发生或者一步也不执行，我们称该组处理步骤为一个事务。当所有的步骤像一个操作一样被完整地执行，我们称该事务被提交。由于其中的一部分或多步执行失败，导致没有步骤被提交，则事务必须回滚到最初的系统状态

事务必须服从ISO/IEC所制定的ACID原则

- ACID是原子性（atomicity）表示事务执行过程中的任何失败都将导致事务所做的任何修改失效
- 一致性（consistency）表示 当事务执行失败时，所有被该事务影响的数据都应该恢复到事务执行前的状态
- 隔离性 （isolation）表示在事务执行过程中对数据的修改，在事务提交之前对其他事务不可见
- 持久性（durability）表示当系统或介质发生故障时，确保已提交事务的更新不能丢失。持久性通过数据库备份和恢复来保证

JDBC 事务是用 Connection 对象控制的。JDBC Connection 接口( java.sql.Connection )提供了两种事务模式：自动提交和手工提交。 java.sql.Connection 提供了以下控制事务的方法： 

~~~java
public void setAutoCommit(boolean) 
public boolean getAutoCommit() 
public void commit() 
public void rollback() 

~~~

使用 JDBC 事务界定时，可以将多个 SQL 语句结合到一个事务中。JDBC 事务的一个缺点是事务的范围局限于一个数据库连接。一个 JDBC 事务不能跨越多个数据库





