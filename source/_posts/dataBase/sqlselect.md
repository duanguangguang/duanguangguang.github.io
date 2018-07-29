---
title: SQL查询
date: 2017-09-01 18:00:30
categories: 
 - 数据库
tags:
 - Oracle
---

## SQL查询

针对常用的SQL查询总结一下

### 一、检索

1. distinct过滤相同的列

   ~~~plsql
   SQL> select distinct column_name from products;--唯一性，distinct作用于所有列
   ~~~

   <!--more -->

2. 限制结果

   - SQL Server和Access

     ~~~plsql
     SQL> select top 5 column_name from products;--检索前五行
     ~~~

   - DB2

     ~~~plsql
     SQL> select column_name from products
     	 fetch first 5 rows only;--检索前五行
     ~~~

   - Oracle

     ~~~plsql
     SQL> select column_name from products
     	 where rownum <= 5;--检索前五行
     ~~~

   - MySQL、MariaDB、PostgreSQL、SQLite

     ~~~plsql
     SQL> select column_name from products
     	 limit 5;--检索前五行
     ~~~

     ~~~plsql
     SQL> select column_name from products
     	 limit 5 offset 5;--检索从第五行起的五行数据
     ~~~

3. 排序数据

   ~~~plsql
   --按列名排
   SQL> select prod_id,prod_price,prod_name
   	 from products
    	 order by prod_price,prod_name;--order by通常位于子句的最后
   ~~~

   ~~~plsql
   --按位置排
   SQL> select prod_id,prod_price,prod_name
   	 from products
    	 order by 2,3;
   ~~~

   ~~~plsql
   --降序
   SQL> select prod_id,prod_price,prod_name
   	 from products
    	 order by prod_price desc,prod_name;--prod_name默认是升序
   ~~~

### 二、过滤

1. where子句过滤

   ~~~plsql
   SQL> select prod_id,prod_price,prod_name
   	 from products
    	 where prod_price between 5 and 10;-->、<、=、is null、<>、!=
   --Microsoft Access 支持<>，不支持!=
   ~~~

2. 求值顺序

   ~~~plsql
   SQL> select prod_id,prod_price,prod_name
   	 from products
    	 where prod_id = 'as' or prod_id = 'bs'
   	 and prod_price > 10;--and 优先于 or
   ~~~

3. in和not

   ~~~plsql
   SQL> select prod_id,prod_price,prod_name
   	 from products
    	 where prod_id in('as','bs') ;--in同or
   ~~~

   ~~~plsql
   SQL> select prod_id,prod_price,prod_name
   	 from products
    	 where not prod_id = 'as';--否定后面的条件
   ~~~

### 三、通配符

~~~plsql
--%,匹配多个
SQL> select prod_name from products where prod_name like 'F%Y';
--Microsoft Access使用的是*而不是%

--_匹配单个
SQL> select prod_name from products where prod_name like 'F_Y';

--[]匹配方括号内任意一个字符
SQL> select prod_name from products where prod_name like '[JM]%';--以J或M开头
~~~

### 四、计算字段

1. 拼接字段

   ~~~plsql
   --+或者||
   SQL> select rtrim(vend_name) + '(' + rtrim(vend_country) + ')'
   	 from vendors
   	 order by vend_name;
   --Assess和SQL Server使用+,DB2、Oracle、postgresql、sqlite、open office base使用||
   --rtrim、ltrim、trim去掉空格
   ~~~

2. 使用别名

   ~~~plsql
   SQL> select rtrim(vend_name) + '(' + rtrim(vend_country) + ')'
   	 as vend_title
   	 from vendors
   ~~~

   MySQL和MariaDB使用的语句：

   ~~~plsql
   SQL> select concat(rtrim(vend_name) + '(' + rtrim(vend_country) + ')')
   	 as vend_title
   	 from vendors
   ~~~

3. 算术运算

   ~~~plsql
   SQL> select prod_id,quantity,price,
   	 quantity*price as expanded
   	 from orderItems;
   ~~~

### 五、函数

1. DBMS函数的差异

   - 提取字符串的组成部分

     > 1. Assess使用mid()
     > 2. DB2、Oracle、postgresql、sqlite使用substr()
     > 3. MySQL和SQL Server使用substring()

   - 数据类型转换

     > 1. Assess和Oracle，每种类型都要转换函数
     > 2. DB2和postgresql使用cast()
     > 3. MariaDB、MySQL和SQL Server使用convert()

   - 取当前时间

     > 1. Assess使用now()
     > 2. DB2和postgresql使用current_date
     > 3. MariaDB和MySQL使用curdate()
     > 4. Oracle使用SYSDATE
     > 5. SQL Server使用getdate()
     > 6. SQLite使用date()

2. soundex

   **soundex**是一个将任何文本串转换为描述其语音表示的字母模式的算法

   ~~~plsql
   --匹配发音类似Mochael green的人员
   SQL> select cust_name,cust_contact
   	 from customers
   	 where soundex(cust_contact) = soundex('Mochael green');
   --Assess和postgresql不支持soundex
   ~~~

3. 日期和时间

   检索2017年的所有订单

   ~~~plsql
   --SQL Server
   SQL> select order_num from orders
   	 where datepart(yy,order_date) = 2017;

   --Access
   SQL> select order_num from orders
   	 where datepart('yyyy',order_date) = 2017;

   --postgresql
   SQL> select order_num from orders
   	 where date_part('year',order_date) = 2017;

   --Oracle
   SQL> select order_num from orders
   	 where to_number(to_char(order_date,'YYYY')) = 2017;

   --MariaDB和MySQL
   SQL> select order_num from orders
   	 where year(order_date) = 2017;

   --SQLite小技巧
   SQL> select order_num from orders
   	 where strftime('%Y',order_date) = '2017';
   ~~~

### 六、汇总

1. 平均值

   ~~~plsql
   SQL> select avg(prod_price) as avg_price from products;
   --avg用于单列，并忽略null值的行
   SQL> select max(prod_price) as avg_price from products;
   SQL> select min(prod_price) as avg_price from products;
   ~~~

2. 计数

   ~~~plsql
   SQL> select count(*) as num_cust from customers;--对所有行计数
   ~~~

3. 汇总

   ~~~plsql
   SQL> select sum(price*quantity) as total
   	 from orderItems;
   ~~~

### 七、分组

1. 创建分组

   ~~~plsql
   SQL> select vend_id,count(*) as num_prods
   	 from products
   	 group by vend_id; --group by 2,1-先按第二列分组，再按第一列分
   ~~~

2. having过滤

   ~~~plsql
   --where可以用having代替，where过滤行，having过滤组
   --having结合group by使用
   SQL> select vend_id,count(*) as num_prods
   	 from products
   	 where vend_price >= 4
   	 group by vend_id
   	 having count(*) >= 2;
   ~~~

3. 排序

   ~~~plsql
   SQL> select vend_id,vend_num,count(*) as num_prods
   	 from products
   	 where vend_price >= 4
   	 group by vend_id
   	 having count(*) >= 2
   	 order by vend_id,vend_num;
   --Access不允许按别名排序
   ~~~

### 八、子查询

1. 子查询

   ~~~plsql
   SQL> select cust_name,cust_contact
   	 from customers
   	 where cust_id 
   	 in(
        	select cust_id
          	from orders
          	where order_num
          	in(
              select order_num
              from orderitems
              where prod_id = 'RGAN01'
           )
        );
   ~~~

2. 子查询作为计算字段

   ~~~plsql
   SQL> select cust_name,cust_state,
   	 	(select count(*) from orders 
         	where orders.cust_id =  customers.cust_id) 
   	 	as ordernum
   	 from customers
   	 order by cust_name;
   ~~~

### 九、联结表

1. 创建联结

   ~~~plsql
   SQL> select vend_name,prod_name,prod_price
   	 from vendors,products
   	 where vendors.vend_id = products.vend_id;
   ~~~

2. 子查询与多表联结

   ~~~plsql
   --多表联结实现上面的子查询
   SQL> select cust_name,cust_contact
   	 from customers,orders,orderitems
   	 where customers.cust_id = orders.cust_id
   	 and orders.order_num = orderitems.order_num
   	 and prod_id = 'RGAN01';
   ~~~

3. 使用表别名

   ~~~plsql
   SQL> select vend_name,prod_name,prod_price
   	 from vendors as v,products as p
   	 where v.vend_id = p.vend_id;
   --oracle中没有as ,直接vendors v
   ~~~

   [更多表联结参考](https://duanguangguang.github.io/2017/08/23/dataBase/sql-join/)

### 十、组合查询

~~~plsql
SQL> select cust_name,cust_contact
	 from customers
	 where cust_state in('ax','as','sd')
	 union
	 select cust_name,cust_contact
	 from customers
	 where cust_name = 'FUNALL'
	 order by cust_name,cust_contact;
--union-取消重复的行，union all不取消重复的行
--组合查询只能有一个order by
~~~

### 十一、表复制

~~~plsql
SQL> select * into custcopy from customers;
--DB2不支持select into

--MariaDB、MySQL、Oracle、postgresql、sqlite
SQL> create table custcopy as
	 select * from customers;
~~~



