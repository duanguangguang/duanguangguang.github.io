---
title: 视图
date: 2017-08-29 19:46:21
categories: 
 - 数据库
tags:
 - Oracle
---

## 视图

视图是一张虚拟表，它表示一张表的部分数据或多张表的综合数据，其结构和数据是建立在对表的查询基础上。视图中并不存放数据，视图只包含使用时动态检索数据的查询。存放在视图所引用的原始表（基表）中同一张原始表，根据不同用户的不同需求，可以创建不同的视图。DBMS支持说明：

- Microsoft Access不支持视图
- MySQL从1.5版本开始支持视图
- SQLite仅支持只读视图，所以视图可以创建，可以读，但不能修改

<!--more -->

### 一、视图的作用

例：检索订购了某种产品的顾客

~~~plsql
SQL> select cust_name, cust_contact
	 from customers, orders, orderitems
	 where customers.cust_id = orders.cust_id
	 and orderitems.order_num = orders.order_num
	 and prod_id = 'rgan01';
~~~

将整个查询包装成一个名为ProductCustomers的虚拟表

~~~plsql
SQL> select cust_name, cust_contact
	 from ProductCustomers
	 where prod_id = 'rgan01';
~~~

ProductCustomers是一个视图，它不包含任何数据，包含的是一个查询。视图提供了一种封装select语句的层次，可以用来简化数据处理，重新格式化或保护基础数据

### 二、使用视图的原因

1. 使用视图的原因

   - 重用SQL
   - 简化复杂的SQL操作，方便重用而不必知道其基本的实现细节
   - 使用表的一部分而不是整个表
   - 保护数据，可以授权用户访问表的特定部分权限，而不是整个表的权限
   - 更改数据格式和表示，视图可返回与底层表的表示和格式不同的数据

2. 视图的使用

   - 创建视图后，可以与表基本相同的方式使用视图（添加和更新存在限制）
   - 视图仅仅是用来查看存储在别处数据的一种设施。视图返回的数据是从其他表中检索出来的，在添加或更改这些表的数据时，视图将返回改变过的数据

3. 性能问题

   > 因为视图不包含数据，所以每次使用视图时，都必须处理查询执行时需要的所有检索。如果在使用中用多个联结和过滤创建了复杂的视图或者嵌套了视图，性能可能会大幅度下降

### 三、视图的规则和限制

一些常用的规则和限制：

- 视图必须唯一命名
- 视图可以嵌套，可以利用从其他视图中检索数据查询来构造视图
- 许多DBMS禁止在视图中使用order by子句
- 对返回的所有列进行命名，计算字段列需要使用别名
- 视图不能索引，也不能有关联的触发器或默认值

### 四、视图的语法

使用T-SQL语句创建视图的语法：

~~~plsql
SQL> CREATE VIEW view_name
AS
<select语句>
~~~

删除视图：

~~~plsql
SQL> drop view view_name;
--覆盖（更新）视图，必须先删除再重新创建
~~~

#### 1. 利用视图简化复杂的联结

创建一个不绑定特点数据的视图：

~~~plsql
SQL> create view productCustomers
	 as 
	 select cust_name,cust_contact,prod_id
	 from customers,orders,orderItems
	 where customers.cust_id = orders.cust_id
	 and orderItems.order_num = orders.order_num;
--联结三个表，返回已订购了任意产品的所有顾客的列表
~~~

使用视图：检索订购了RGAN01的顾客

~~~plsql
SQL> select cust_name,cust_contact
	 from productCustomers
	 where prod_id = 'RGAN01';
--利用视图，可一次性编写好基础的SQL，然后根据需要多次调用
~~~

#### 2. 用视图重新格式化检索出来的数据

在单个组合计算列中返回供应商名和位置：

~~~plsql
SQL> select rtrim(vend_name) + '(' + rtrim(vend_country) + ')'--+同 ||,MySQL使用concet函数
	 as vend_title
 	 from vendors
	 order by vend_name;
~~~

如果经常需要这个格式的结果，可以将这个转成视图：

~~~plsql
SQL> create view vendorLocations
	 as 
	 select rtrim(vend_name) + '(' + rtrim(vend_country) + ')'--+同 ||,MySQL使用concet函数
	 as vend_title
 	 from vendors
~~~

利用该视图检索数据：

~~~plsql
SQL> select * from vendorLocations;
~~~

#### 3. 用视图过滤数据

过滤没有电子邮件地址的顾客：

~~~plsql
SQL> create view customerEMailList
	 as
	 select cust_id,cust_name,cust_email
	 from Customers
	 where cust_email is not null;
~~~

使用该视图过滤：

~~~plsql
SQL> select * from customerEMailList;
--视图中的where子句和使用视图检索的where子句会自动组合
~~~

#### 4. 用视图与计算字段

检索某个订单中的物品，计算每种物品的总价格：

~~~plsql
SQL> select prod_id,quantity,item_price,quantity*item_price as expanded_price
	 from OrderItems
	 where order_num = abc;
~~~

转换成视图：

~~~plsql
SQL> create view orderItemsExpanded
	 as
	 select prod_id,quantity,item_price,quantity*item_price as expanded_price
	 from OrderItems
~~~

检索abc的详细内容：

~~~plsql
SQL> select * from orderItemsExpanded
	 where order_num = abc;
~~~

