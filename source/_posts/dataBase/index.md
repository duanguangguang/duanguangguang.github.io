---
title: 索引基本概念
date: 2017-08-29 19:55:17
categories: 
 - 数据库
tags:
 - Sql优化
---

## 索引

索引用来排序数据以加快搜索和排序操作的速度。使用索引，可以在一个或多个列上定义索引，使DBMS保存其内容的一个排过序的列表。在定义了索引后，DBMS搜索排过序的索引，找出匹配的位置，然后检索这些行。

创建索引语法：

~~~plsql
SQL> create [unique][clustered|nonclustered]index index_name --索引名必须唯一
	 on {table_name|view_name}[with [index_property]];--on指定被索引的表，括号中指明索引包含的列
--例：建立唯一索引
SQL> create unique index emp_email on employees(email) tablespace users;
~~~

说明：**unique**：建立唯一索引，**clustered**：建立聚集索引，**nonclustered**：建立非聚集索引，**index_property**：索引属性。unique索引既可以采用聚集索引结构，也可以采用非聚集索引的结构。

删除索引语法：

~~~plsql
SQL> drop index table_name.index_name[,table_name.index_name]--索引所在的表名称.索引名称
~~~

显示索引信息：

~~~plsql
--oracle用user_indexes和user_ind_columns系统表查看已经存在的索引
--user_indexes: 系统视图存放是索引的名称以及该索引是否是唯一索引等信息。
--user_ind_column: 系统视图存放的是索引名称，对应的表和列等。
--查看索引个数和类别:
SQL> select * from user_indexes where table='表名';
--查看索引被索引的字段:
SQL> select * from user_ind_columns where index_name=upper('&index_name');
--通过类似下面的语句来查看一个表的索引的基本情况：
SQL> select user_ind_columns.index_name,user_ind_columns.column_name,
	 user_ind_columns.column_position,user_indexes.uniqueness
	 from user_ind_columns,user_indexes
	 where user_ind_columns.index_name = user_indexes.index_name
	 and user_ind_columns.table_name = '你想要查询的表名字';

--SQL Server使用系统存储过程：sp_helpindex 查看指定表的索引信息。
SQL> Exec sp_helpindex book1;
~~~

<!--more -->

### 一、 索引的类型

1. 唯一索引：唯一索引不允许两行具有相同的索引值
2. 主键索引：为表定义一个主键将自动创建主键索引，主键索引是唯一索引的特殊类型。主键索引要求主键中的每个值是唯一的，并且不能为空
3. 聚集索引(Clustered)：表中各行的物理顺序与键值的逻辑（索引）顺序相同，每个表只能有一个
4. 非聚集索引(Non-clustered)：非聚集索引指定表的逻辑顺序。数据存储在一个位置，索引存储在另一个位置，索引中包含指向数据存储位置的指针。可以有多个，小于249个

举例：

1. 图书馆对图书的管理建立一个以字母开头的目录，例如：a开头的书，在第一排，b开头的在第二排，这样在找什么书就好说了，这个就是一个**聚集索引**。很多人借书找某某作者的，不知道书名怎么办？图书管理员在写一个目录，某某作者的书分别在第几排，第几排，这就是一个**非聚集索引**。
2. 字典前面的目录，可以按照拼音和部首去查询，我们想查询一个字，只需要根据拼音或者部首去查询，就可以快速的定位到这个汉字了，这个就是索引的好处，拼音查询法就是**聚集索引**，部首查询就是一个**非聚集索引**。

### 二、索引的存储机制

首先，无索引的表，查询时，是按照顺序存续的方法扫描每个记录来查找符合条件的记录，这样效率十分低下。 聚集索引和非聚集索引的根本区别是表记录的排列顺序和与索引的排列顺序是否一致，其实理解起来非常简单，还是举字典的例子：如果按照拼音查询，那么都是从a-z的，是具有连续性的，a后面就是b，b后面就是c， 聚集索引就是这样的，他是和表的物理排列顺序是一样的，例如有id为聚集索引，那么1后面肯定是2，2后面肯定是3，所以说这样的搜索顺序的就是聚集索引。非聚集索引就和按照部首查询是一样是，可能按照偏房查询的时候，根据偏旁‘弓’字旁，索引出两个汉字，张和弘，但是这两个其实一个在100页，一个在1000页，（这里只是举个例子），他们的索引顺序和数据库表的排列顺序是不一样的，这个样的就是非聚集索引。

所以他们的存储机制是这样的：聚集索引就是在数据库被开辟一个物理空间存放他的排列的值，例如1-100，所以当插入数据时，他会重新排列整个整个物理空间，而非聚集索引其实可以看作是一个含有聚集索引的表，他只仅包含原表中非聚集索引的列和指向实际物理表的指针。他只记录一个指针，其实就有点和堆栈差不多的感觉了。

### 三、索引建立原则

|        动作描述        | 使用聚集索引 | 使用非聚集索引 |
| :----------------: | :----: | :-----: |
|        外键列         |   应    |    应    |
|        主键列         |   应    |    应    |
| 列经常被分组排序（order by） |   应    |    应    |
|     返回某范围内的数据      |   应    |   不应    |
|      小数目的不同值       |   应    |   不应    |
|      大数目的不同值       |   不应   |    应    |
|       频繁更新的列       |   不应   |    应    |
|      频繁修改索引列       |   不应   |    应    |
|      一个或极少不同值      |   不应   |   不应    |

1.  定义主键的数据列一定要建立索引；定义有外键的数据列一定要建立索引。
2. 对于经常查询的数据列最好建立索引。
3. 对于需要在指定范围内的快速或频繁查询的数据列。
4. 经常用在WHERE子句中的数据列。
5. 经常出现在关键字order by、group by、distinct后面的字段，建立索引。如果建立的是复合索引，索引的字段顺序要和这些关键字后面的字段顺序一致，否则索引不会被使用。
6. 对于那些查询中很少涉及的列，重复值比较多的列不要建立索引。
7. 对于定义为text、image和bit的数据类型的列不要建立索引。
8. 对于经常存取的列避免建立索引 。
9.  限制表上的索引数目。对一个存在大量更新操作的表，所建索引的数目一般不要超过3个，最多不要超过5个。索引虽说提高了访问速度，但太多索引会影响数据的更新操作。
10. 对复合索引，按照字段在查询条件中出现的频度建立索引。在复合索引中，记录首先按照第一个字段排序。对于在第一个字段上取值相同的记录，系统再按照第二个字段的取值排序，以此类推。因此只有复合索引的第一个字段出现在查询条件中，该索引才可能被使用,因此将应用频度高的字段，放置在复合索引的前面，会使系统最大可能地使用此索引，发挥索引的作用。

### 四、索引的优缺点

- 优点：

  > 1. 加快访问速度
  > 2. 加强行的唯一性

- 缺点：

  > 1. 带索引的表在数据库中需要更多的存储空间
  > 2. 操纵数据的命令需要更长的处理时间，因为它们需要对索引进行更新

### 五、索引注意事项

1. 索引改善检索操作的性能，但降低了数据插入、修改和删除的性能。在执行这些操作时，DBMS必须动态地更新索引
2. 索引数据需要占用大量的存储空间
3. 索引用于数据过滤和数据排序，如果经常需要以某种特定顺序排序数据，该数据适合做索引
4. 可以在索引中定义多个列（如：州加城市）。这样索引仅在以州加城市的顺序排序时有用
5. 索引的效率随表数据的改变而变化，所以需要定期检查索引，使其性能达到最好

### 六、索引覆盖

假设你在Sales表(SelesID,SalesDate,SalesPersonID,ProductID,Qty)的外键列(ProductID)上创建了一个索引，假设ProductID列是一个高选中性列，那么任何在where子句中使用索引列(ProductID)的select查询都会更快，如果在外键上没有创建索引，将会发生全部扫描，但还有办法可以进一步提升查询性能。假设Sales表有10,000行记录，下面的SQL语句选中400行(总行数的4%)：　

~~~plsql
SELECT SalesDate, SalesPersonID FROM Sales WHERE ProductID = 112
~~~

我们来看看这条SQL语句在SQL执行引擎中是如何执行的：

1. Sales表在ProductID列上有一个非聚集索引，因此它查找非聚集索引树找出ProductID=112的记录；
2. 包含ProductID = 112记录的索引页也包括所有的聚集索引键(所有的主键键值，即SalesID)；
3. 针对每一个主键(这里是400)，SQL引擎查找聚集索引树找出真实的行在对应页面中的位置；
4. SQL引擎从对应的行查找SalesDate和SalesPersonID列的值。

在上面的步骤中，对ProductID = 112的每个主键记录(这里是400)，SQL引擎要搜索400次聚集索引树以检索查询中指定的其它列(SalesDate，SalesPersonID)。如果非聚集索引页中包括了聚集索引键和其它两列(SalesDate,，SalesPersonID)的值，SQL Server引擎可能不会执行上面的第3和4步，直接从非聚集索引树查找ProductID列速度还会快一些，直接从索引页读取这三列的数值。

幸运的是，有一种方法实现了这个功能，它被称为“覆盖索引”，在表列上创建覆盖索引时，需要指定哪些额外的列值需要和聚集索引键值(主键)一起存储在索引页中。下面是在Sales 表ProductID列上创建覆盖索引的例子：　

~~~plsql
CREATE INDEX NCLIX_Sales_ProductID--Index name
　　ON dbo.Sales(ProductID)--Column on which index is to be created
　　INCLUDE(SalesDate, SalesPersonID)--Additional column values to include
~~~

应该在那些select查询中常使用到的列上创建覆盖索引，但覆盖索引中包括过多的列也不行，因为覆盖索引列的值是存储在内存中的，这样会消耗过多内存，引发性能下降。

###  七、索引碎片

[数据库自身优化]([http://www.cnblogs.com/AK2012/archive/2012/12/25/2012-1228.html](http://www.cnblogs.com/AK2012/archive/2012/12/25/2012-1228.html))

### 八、索引实战（摘抄）

摘抄地址：[如何使你的SQL运行得更快]([http://blog.csdn.net/gprime/article/details/1687930](http://blog.csdn.net/gprime/article/details/1687930))

人们在使用SQL时往往会陷入一个误区，即太关注于所得的结果是否正确，而忽略了不同的实现方法之间可能存在的性能差异，这种性能差异在大型的或是复杂的数据库环境中（如联机事务处理OLTP或决策支持系统DSS）中表现得尤为明显。而不良的SQL往往来自于不恰当的索引设计、不充份的连接条件和不可优化的where子句。

#### 不合理的索引设计

例：表record有620000行，试看在不同的索引下，下面几个 SQL的运行情况：

1. 在date上建有一非聚集索引

   ~~~plsql
   select count(*) from record where date >'19991201' and date < '19991214'and amount >2000 (25秒)
   select date ,sum(amount) from record group by date(55秒)
   select count(*) from record where date >'19990901' and place in ('BJ','SH') (27秒)
   ~~~

   分析：date上有大量的重复值，在非聚集索引下，数据在物理上随机存放在数据页上，在范围查找时，必须执行一次表扫描才能找到这一范围内的全部行。

2. 在date上建有一聚集索引

   ~~~plsql
   select count(*) from record where date >'19991201' and date < '19991214' and amount >2000 （14秒）
   select date,sum(amount) from record group by date（28秒）
   select count(*) from record where date >'19990901' and place in ('BJ','SH')（14秒）
   ~~~

   分析：在聚集索引下，数据在物理上按顺序在数据页上，重复值也排列在一起，因而在范围查找时，可以先找到这个范围的起末点，且只在这个范围内扫描数据页，避免了大范围扫描，提高了查询速度。

3. 在place，date，amount上的组合索引

   ~~~plsql
   select count(*) from record where date >'19991201' and date < '19991214' and amount >2000 （26秒）
   select date,sum(amount) from record group by date（27秒）
   select count(*) from record where date >'19990901' and place in ('BJ, 'SH')（< 1秒）
   ~~~

   分析：这是一个不很合理的组合索引，因为它的前导列是place，第一和第二条SQL没有引用place，因此也没有利用上索引；第三个SQL使用了place，且引用的所有列都包含在组合索引中，形成了索引覆盖，所以它的速度是非常快的。

4. 在date，place，amount上的组合索引

   ~~~plsql
   select count(*) from record where date >'19991201' and date < '19991214' and amount >2000(< 1秒)
   select date,sum(amount) from record group by date（11秒）
   select count(*) from record where date >'19990901' and place in ('BJ','SH')（< 1秒）
   ~~~

   分析：这是一个合理的组合索引。它将date作为前导列，使每个SQL都可以利用索引，并且在第一和第三个SQL中形成了索引覆盖，因而性能达到了最优。

总结：

缺省情况下建立的索引是非聚集索引，但有时它并不是最佳的；合理的索引设计要建立在对各种查询的分析和预测上。

一般来说：

1. 有大量重复值、且经常有范围查询（between, >,< ，>=,< =）和order by、group by发生的列，可考虑建立聚集索引。
2. 经常同时存取多列，且每列都含有重复值可考虑建立组合索引。
3. 组合索引要尽量使关键查询形成索引覆盖，其前导列一定是使用最频繁的列。

#### 不充分的连接条件

例：表card有7896行，在card_no上有一个非聚集索引，表account有191122行，在account_no上有一个非聚集索引，试看在不同的表连接条件下，两个SQL的执行情况：

~~~plsql
select sum(a.amount) from account a,card b where a.card_no = b.card_no（20秒）
select sum(a.amount) from account a,card b where a.card_no = b.card_no 
	   and a.account_no=b.account_no（< 1秒）
~~~

分析：

在第一个连接条件下，最佳查询方案是将account作外层表，card作内层表，利用card上的索引，其I/O次数可由以下公式估算为：外层表account上的22541页+（外层表account的191122行*内层表card上对应外层表第一行所要查找的3页）=595907次I/O。

在第二个连接条件下，最佳查询方案是将card作外层表，account作内层表，利用account上的索引，其I/O次数可由以下公式估算为：外层表card上的1944页+（外层表card的7896行*内层表account上对应外层表每一行所要查找的4页）= 33528次I/O。

可见，只有充份的连接条件，真正的最佳方案才会被执行。

总结：

1. 多表操作在被实际执行前，查询优化器会根据连接条件，列出几组可能的连接方案并从中找出系统开销最小的最佳方案。连接条件要充份考虑带有索引的表、行数多的表；内外表的选择可由公式：**外层表中的匹配行数*内层表中每一次查找的次数确定**，乘积最小为最佳方案。
2. 查看执行方案的方法-- 用set showplanon，打开showplan选项，就可以看到连接顺序、使用何种索引的信息；想看更详细的信息，需用sa角色执行dbcc(3604,310,302)。（数据库：Sybase11.0.3）

#### 不可优化的where子句

1. 例：下列SQL条件语句中的列都建有恰当的索引，但执行速度却非常慢：

   ~~~plsql
   select * from record where substring(card_no,1,4)='5378'(13秒)
   select * from record where amount/30< 1000（11秒）
   select * from record where convert(char(10),date,112)='19991201'（10秒）
   ~~~

   分析：where子句中对列的任何操作结果都是在SQL运行时逐列计算得到的，因此它不得不进行表搜索，而没有使用该列上面的索引；如果这些结果在查询编译时就能得到，那么就可以被SQL优化器优化，使用索引，避免表搜索，因此将SQL重写成下面这样：你会发现SQL明显快起来！

   ~~~plsql
   select * from record where card_no like'5378%'（< 1秒）
   select * from record where amount< 1000*30（< 1秒）
   select * from record where date= '1999/12/01'（< 1秒）
   ~~~

2. 例：表stuff有200000行，id_no上有非群集索引，请看下面这个SQL：

   ~~~plsql
   select count(*) from stuff where id_no in('0','1')（23秒）
   ~~~

   分析：where条件中的'in'在逻辑上相当于'or'，所以语法分析器会将in ('0','1')转化为id_no ='0' or id_no='1'来执行。

   我们期望它会根据每个or子句分别查找，再将结果相加，这样可以利用id_no上的索引；

   但实际上（根据showplan）,它却采用了"OR策略"，即先取出满足每个or子句的行，存入临时数据库的工作表中，再建立唯一索引以去掉重复行，最后从这个临时表中计算结果。因此，实际过程没有利用id_no上索引，并且完成时间还要受tempdb数据库性能的影响。

   实践证明，表的行数越多，工作表的性能就越差，当stuff有620000行时，执行时间竟达到220秒！还不如将or子句分开：

   ~~~plsql
   select count() from stuff where id_no='0'
   select count() from stuff where id_no='1'
   ~~~

   得到两个结果，再作一次加法合算。因为每句都使用了索引，执行时间只有3秒，在620000行下，时间也只有4秒。

   或者，用更好的方法，写一个简单的存储过程：

   ~~~plsql
   create proc count_stuff asdeclare @a intdeclare @b intdeclare @c intdeclare @d char(10)beginselect @a=count() from stuff where id_no='0'select @b=count() from stuff where id_no='1'endselect @c=@a+@bselect @d=convert(char(10),@c)print @d
   ~~~

   直接算出结果，执行时间同上面一样快！

总结：所谓优化即where子句利用了索引，不可优化即发生了表扫描或额外开销。

1. 任何对列的操作都将导致表扫描，它包括数据库函数、计算表达式等等，查询时要尽可能将操作移至等号右边。
2. in、or子句常会使用工作表，使索引失效；如果不产生大量重复值，可以考虑把子句拆开；拆开的子句中应该包含索引。
3. 要善于使用存储过程，它使SQL变得更加灵活和高效。

从以上这些例子可以看出，SQL优化的实质就是在结果正确的前提下，用优化器可以识别的语句，充份利用索引，减少表扫描的I/O次数，尽量避免表搜索的发生。其实SQL的性能优化是一个复杂的过程，上述这些只是在应用层次的一种体现，深入研究还会涉及数据库层的资源配置、网络层的流量控制以及操作系统层的总体设计。

