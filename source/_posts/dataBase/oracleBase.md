---
title: Oracle基础
date: 2017-08-24 22:09:55
categories: 
 - 数据库
tags:
 - Oracle
---

## Oracle基础

- 用户登录

  ~~~plsql
  --使用用户名/密码登陆
  SQL>connect username/password;
  --使用sys登陆，权限最高
  SQL>connect sys/password as sysdba;
  ~~~

- 查看登陆用户

  ~~~plsql
  SQL>show user;
  --dba_users用户信息的数据字典
  SQL>select username from dba_users;
  ~~~

- 启用scott用户

  ~~~plsql
  SQL>alter user scott account unlook;
  SQL>connect scott/password;
  ~~~

  <!-- more -->

### 一、表空间

**表空间**是数据库的逻辑存储空间，在数据库中开辟的空间用来存储数据对象。**表空间**由数据文件构成，数据库可以由多个表空间来构成

1. 永久表空间

   表、视图、存储过程

2. 临时表空间

   中间过程、临时表

3. UNDO表空间（安度）

   保存事物所修改的旧值，可以执行撤销操作

4. 设置默认表空间

   ~~~plsql
   --设置user为默认表空间
   SQL>alter user system DEFAULT TABLESPACE system;
   ~~~

5. 创建表空间

   ~~~plsql
   SQL>create tablespace TEST_TABLESPACE datafile 'testfile.dbf' size 10m;
   --创建临时表空间
   SQL>create temporary tablespace TEMP tempfile 'tempfile.dbf' size 10m;
   ~~~

6. 查看表空间

   ~~~plsql
   SQL>desc dba_tablespaces;--user_tablespace
   SQL>select tablespace_name from dba_tablespace;
   SQL>select file_name from dba_data_files where tablespace_name = 'TEST_TABLESPACE';
   ~~~

7. 修改表空间

   ~~~plsql
   --设置联机或脱机状态
   SQL>alter tablespace TEST_TABLESPACE online;--offline
   --查看表空间状态
   SQL>select status from dba_tablespace where tablespace_name = 'TEST_TABLESPACE';
   --设置只读或可读可写状态（联机状态下）
   SQL>alter tablespace TEST_TABLESPACE read only;--read write
   ~~~

8. 修改数据文件

   ~~~plsql
   --增加数据文件
   SQL>alter tablespace TEST_TABLESPACE add datafile 'testfile2.dbf' size 10m;
   --查看数据文件
   SQL>select file_name from dba_data_files where tablespace_name = 'TEST_TABLESPACE';
   --删除数据文件
   SQL>alter tablespace TEST_TABLESPACE drop datafile 'testfile2.dbf';
   --注：不能删除创建表空间时创建的数据文件，否则需要删除该表空间
   ~~~

9. 删除表空间

   ~~~plsql
   --只删除表空间
   SQL>drop tablespace TEST_TABLESPACE;
   --删除表空间及数据
   SQL>drop tablespace TEST_TABLESPACE including contients;
   ~~~

### 二、管理表

#### 1. 认识表

- 表是基本存储单位，位于表空间

- 二维结构：行----记录；列---域或字段

- 约定：

  > 1. 每一列数据必须具有相同数据类型
  > 2. 列名唯一
  > 3. 每一行数据的唯一性

#### 2. 数据类型（oracle 11g）

1. 字符型

   - 固定长度

     ~~~plsql
     --按照unicode编码，存汉字情况多
     CHAR(n);--max:2000
     NCHAR(n);--max:1000
     ~~~

   - 可变长度

     ~~~plsql
     VARCHAR2(n);--max:4000
     NVARCHAR2(n);--max:2000
     ~~~

2. 数值型

   ~~~plsql
   NUMBER(p,s);--p表示有效数字，s表示小数点后的位数。例：NUMBER(5,2) ---123.45

   FLOAT(n);--主要存储二进制类型，能存储二进制位数1-126位
   --二进制转十进制：给这个数乘以0.30103
   ~~~

3. 日期型

   ~~~plsql
   DATE--表示范围：公元前4712年1月1日到公元9999年12月31日

   TIMESTAMP--时间戳，可精确到小数秒
   ~~~

4. 其他类型（存放大数据）

   ~~~plsql
   BLOB--可存放4GB数据，以二进制形式存
   CLOB--可存放4GB数据，以字符串形式存
   ~~~

#### 3. 管理表

1. 创建表

   ~~~plsql
   create table table_name
   (
      column_name datatype, ...
   )
   ~~~

2. 添加字段

   ~~~plsql
   alter table table_name add column_name datatype;
   ~~~

3. 更改字段数据类型

   ~~~plsql
   alter table table_name modify column_name datatype;
   ~~~

4. 删除字段

   ~~~plsql
   alter table table_name drop column column_name;
   ~~~

5. 修改字段名

   ~~~plsql
   alter table table_name rename column column_name to new_column_name;
   ~~~

6. 修改表名

   ~~~plsql
   rename table_name to new_table_name;
   ~~~

7. 删除表

   ~~~plsql
   delete from table_name where column_name = column_value;--删除表中数据
   truncate table table_name;--删除表中数据，效果比delete快很多，也称为截断
   drop table table_name;--删除表结构
   ~~~

### 三、操作表中数据

#### 1. 添加

~~~plsql
insert into table_name (column1,column1,...) values (value1,value2,...)
--默认值：default,创建表修改表都可以使用
~~~

#### 2. 复制表数据

~~~plsql
--创建表时复制
create table table_name
as
select column1,...|* from table_old
--添加时复制
insert into table_name
[(column1,...)] --复制全部数据时可省略
select column1,...|* from table_old
~~~

#### 3. 修改数据

~~~plsql
update table_name set column1 = value1,...
[where conditions]
~~~

#### 4. 删除数据

~~~plsql
delete from table_name
[where conditions]
~~~

### 四、约束

约束作用：定义规则，确保数据完整性

#### 1. 非空约束

- 创建表时设置非空约束

  ~~~plsql
  create table table_name
  (
     column_name datatype not null,
    	...
  )
  ~~~

- 修改表时设置非空约束

  ~~~plsql
  --前提时修改的表无数据。可先delete再修改
  alter table table_name
  modify column_name datatype not null
  ~~~

#### 2. 主键约束

确保表当中每一行数据的唯一性，非空。一张表只能设计一个主键约束。主键约束可以由多个字段构成（联合主键/复合主键）

1. 创建表时创建

   ~~~plsql
   create table table_name
   (
      column_name datatype primary key,
     	...
   )
   ~~~

2. 联合主键举例

   ~~~plsql
   create table userinfo
   (  id number(6,0),
      username varchar2(20),
      userpass varchar2(20),
      constraint pk_id_username primary key(id,username)
      --pk_id_username是主键约束名
   )
   ~~~

3. 查主键

   ~~~plsql
   --user_constraints用户数据字典
   select constraint_name from user_constraints where table_name = 'userinfo';
   ~~~

4. 修改表时添加主键约束

   ~~~plsql
   alter table table_name
   add constraint constraint_name --主键名
   primary key(column_name1,...);
   ~~~

5. 更改约束名

   ~~~plsql
   alter table userinfo
   rename constraint pk_id to new_pk_id;
   ~~~

6. 删除主键约束

   - 禁用

     ~~~plsql
     alter table userinfo disable|enable constraint constraint_name;--禁用/启用
     --查看禁用启用
     select constraint_name,status from user_constraints where table_name = 'userinfo';
     ~~~

   - 删除

     ~~~plsql
     alter table userinfo drop constraint constraint_name;
     ~~~

   - 涉及外键

     ~~~plsql
     --涉及外键时，将引用该主键的也删掉
     alter table table_name drop primary key [cascade]
     ~~~

#### 3. 外键约束

注：

- 设置外键约束时，主表的字段必须是主键
- 主从表中相应的字段必须是同一个数据类型
- 从表中外键字段的值必须来自主表中相应字段的值或者为null值

1. 创表时---列及

   ~~~plsql
   create table table1 --从表
   (
      column_name datatype references
      table2(column_name) --主表
     	...
   )
   ~~~

2. 创表时---表及

   ~~~plsql
   constraint constraint_name foreign key (column_name)
   references table_name (column_name)[on delete cascade] --及联删除所在行
   ~~~

3. 例：

   ~~~plsql
   create table userinfo_f2
   (  id varchar2(10) primary key,
      username varchar2(20),
      typeid_new varchar2(10) references typeinfo(typeid) --列及
       ...
      typeid_new_t2 varchar2(10),
      constraint fk_typeid_new foreign key (typeid_new_t2) 
      references typeinfo (typeid) --表及
   )
   ~~~

4. 修改表时设置外键约束

   ~~~plsql
   alter table table_name
   add constraint constraint_name foreign key (column_name)
   references table_name (column_name)[on delete cascade]
   ~~~

5. 删除外键约束

   - 禁用

     ~~~plsql
     alter table table_name
     disable|enable constraint constraint_name
     ~~~

   - 删除

     ~~~plsql
     alter table table_name
     drop constraint constraint_name
     ~~~

#### 4. 唯一约束

保证字段值的唯一性

1. 唯一约束与主键约束的区别

   |      | 主键约束 | 唯一约束         |
   | ---- | ---- | ------------ |
   | 是否可空 | 非空   | 可空           |
   | 值个数  | 一个   | 可多个，但空值只能有一个 |

2. 创建表设置唯一约束

   - 列及

     ~~~plsql
     create table table_name
     (
       column_name datatype unique,
       ...
     )
     ~~~

   - 表及

     ~~~plsql
     create table table_name
     (
       column_name datatype,
       constraint constraint_name unique (column_name)
     )
     ~~~

3. 修改表时添加唯一约束

   ~~~plsql
   alter table table_name
   add constraint constraint_name unique (column_name)
   ~~~

4. 删除唯一约束

   - 禁用

     ~~~plsql
     alter table table_name
     disable|enable constraint constraint_name
     ~~~

   - 删除

     ~~~plsql
     alter table table_name
     drop constraint constraint_name
     ~~~

#### 5. 检查约束

使数据有意义

1. 创表时设置检查约束

   - 列及

     ~~~plsql
     create table table_name
     (
       column_name datatype check (expressions),
       ...
     )
     --例：
     create table userinfo_f3
     (
       salary number(5,0) check (salary > 0)
     )
     ~~~

   - 表及

     ~~~plsql
     create table table_name
     (
       column_name datatype,
       constraint constraint_name check (expressions)
     )
     ~~~

     ​

2. 修改表时添加检查约束

   ~~~plsql
   alter table table_name
   add constraint constraint_name check (expressions)
   ~~~

3. 查看约束名

   ~~~plsql
   select constraint_name,constraint_type,status from user_constraints where table_name = 'userinfo_f3';
   ~~~

4. 删除检查约束

   - 禁用

     ~~~plsql
     alter table table_name
     disable|enable constraint constraint_name
     ~~~

   - 删除

     ~~~plsql
     alter table table_name
     drop constraint constraint_name
     ~~~

### 五、查询

#### 1. 基本查询语句

~~~plsql
select [distinct] column_name1, ... | *
from table_name [where conditions]
~~~

#### 2. 在sql*plus中设置格式

~~~plsql
--设置结果的字段名
column column_name heading new_name;
--设置格式
column column_name format dataformat;
  --注：字符型只能设置长度（a开头）
  column username format a10;--将长度改为10
  --注：数值型可以用9代表一位数字
  column salary format 9999.9;--还可以$99.9
--清除格式
column column_name clear;
~~~

#### 3. 给字段设置别名

针对查询结果，不改变表的列名

~~~plsql
select column_name as new_name, ... from table_name;
~~~

#### 4. 运算符和表达式

| 算术运算符 | +、-、*、/        |
| ----- | -------------- |
| 比较运算符 | >、>=、<、<=、=、<> |
| 逻辑运算符 | and、or、not     |
| 优先级   | not>and>or；比>算 |

例：

~~~plsql
select salary+200 from salarys;
select username from salarys where salary > 800 and salary <> 1800;
~~~

#### 5. 模糊查询

使用通配符**_**或**%**，一个**_**只能代表一个字符，%可以代表0到多个任意字符。例：

~~~plsql
select * from users where username like 'a%';--以a开头
select * from users where username like '_a';--第二个字母是a
select * from users where username like '%a%';--含有a
~~~

#### 6. 范围查询

范围查询使用**in/not in**或**between and**。例：

~~~plsql
select * from users where salary between 800 and 2000;--[800,2000]
select * from users where username in ('as','ad');
~~~

#### 7. 查询结果排序

~~~plsql
select column_name from table_name where [coditions]
order by column_name desc|asc --降|升
~~~

#### 8. case...when语句

~~~plsql
--用法1
case column_name
when value1 then result1,
...
[else resultn]
end
 --例：
 select username,case username when 'aaa' then '计算机部门'
 when 'bbb' then '市场部'
 else '其他部门' end as 部门;

--用法2:case搜索的行径
case
when column_name = value1
then result1, ... [else resultn] end
 --例：
 select username,case when salary < 800 then '工资低'
 when salary > 5000 then '工资高'
 end as 工资水平 from users;
~~~

#### 9. decode函数的使用

~~~plsql
decode (column_name,value1,result1,...,defaultvalue)
--例：
select username,decode (username,'aaa','计算机部','bbb','市场部','其他')
as 部门 from users;
~~~