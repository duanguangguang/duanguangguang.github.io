---
title: 触发器
date: 2017-08-28 16:53:24
categories: 
 - 数据库
tags:
 - Oracle
---

## Oracle触发器

数据库**触发器**是一个与表相关联的、存储的PL/SQL程序。触发器作用：每当一个特定的数据操作语句（insert、update、delete）在指定的表上发出时，oracle自动地执行触发器中定义的语句序列。例：每当成功插入新员工后自动打印“成功插入新员工”

~~~plsql
SQL> create trigger saysuccemp
	 after insert
	 on emp
	 declare
	 begin
		dbms_output.put_line('成功插入新员工');
	 end;
	 /
--插入数据库操作
SQL> insert into emp(empno,ename,sal,deptno)
	 values
	 (1001,'Tom',3000,10);
~~~

<!-- more -->

###  一、触发器语法

~~~plsql
SQL> create [or replace] trigger 触发器名
	 before | after
	 delete | insert | update [of 列名] --只有当更新该列名时才出发
	 on 表名
	 [for each row [where(条件)]] --指明触发器的类型，行级触发器
	 PLSQL块
~~~

- 语句级触发器

  在指定的操作语句操作之前或之后执行一次，不管这条语句影响了多少行

- 行级触发器

  触发语句作用的每一条记录都被触发。在行级触发器总使用 : old 和 : new伪记录变量，识别值的状态

- 区别

  ~~~plsql
  SQL> insert into emps select * from emp where deptno = 10;--emp中有3条记录
  --语句级触发器：针对的是表，所以该插入操作只会调用该触发器一次
  --行级触发器：针对的是行，所以该插入操作会调用该触发器三次
  ~~~

### 二、触发器的应用

1. 复杂的安全性检查

   > 例：非工作时间禁止操作数据库中的数据

   ~~~plsql
   /*
   1.针对的是表，所以采用数据级触发器
   2.周末：to_char(sysdate,'day') in ('星期六','星期天');
   3.上班前，下班后：to_number(to_char(sysdate,'hh24')) not between 9 and 18;
   */
   SQL> create or replace trigger securityemp
   	 before insert
   	 on emp
   	 begin
   		if to_char(sysdate,'day') in ('星期六','星期天') or
   		   to_number(to_char(sysdate,'hh24')) not between 9 and 18
   		then
   		   raise_application_error(-20001,'非工作时间禁止操作数据库中的数据');
   		   --这里表示应用层错误，错误代码范围[-20000,-20999]
   		   --不能用raise抛出例外，raise表示数据库错误
   		end if;
   	 end;
   	 /
   SQL> insert into emp(empno,ename,sal,deptno)
   	 values
   	 (1001,'Tom',3000,10);
   ~~~

2. 数据的确认

   > 例：只有当涨后的工资大于涨前的工资时，才执行涨工资操作

   ~~~plsql
   /*
   检查记录行，所以是行级触发器
   */
   SQL> create or replace trigger checksalary
   	 before update
   	 on emp
   	 for each row --由于每行都要检查所以没有where条件
   	 begin
   		if :new.sal < :old.sal
   		then
   		   raise_application_error(-20002,'涨后的工资不能小于涨前');
   		end if;
   	 end;
   	 /
   SQL> update emp set sal = sal + 1 where empno = 7839;
   	 update emp set sal = sal - 1 where empno = 7839;
   ~~~

3. 数据库的审计

   > 1. 跟踪表上所做数据的操作，什么时间，什么人操作了什么数据
   > 2. oracle已经实现了数据库的审计，共有5种，基于触发器的只是其中一种，这种叫基于值的审计

   ~~~plsql
   /*
   1.给员工涨工资，当涨后的薪水超过6000块时，审计该员工的信息
   2.行级触发器
   */

   --创建表用来保存审计信息
   SQL> create table audit_info(information varchar2(200));

   SQL> create or replace trigger do_audit_emp_salary
   	 after update
   	 on emp
   	 for each row 
   	 begin
   		if :new.sal > 6000;
   		then
   		   insert into audit_info values(:new.empno||' '||:new.ename||' '||:new.sal);
   		end if;
   	 end;
   	 /
   SQL> update emp set sal = sal + 2000;

   --information
   --7839  Tom  6100
   ~~~

4. 数据库的备份与同步

   > 当主数据库的数据更改，通过触发器将数据同步到从数据库中作备份

   ~~~plsql
   /*
   1.给员工涨工资后，自动备份新的工资到备份表中
   2.行级触发器
   */

   --创建备份表
   SQL> create table emp_back as select * from emp;

   SQL> create or replace trigger sync_salary
   	 after update
   	 on emp
   	 for each row 
   	 begin
   		--当主表更新后，自动更新备份表
   		update emp_back set sal = :new.sal where empno = :new.empno;
   	 end;
   	 /
   SQL> update emp set sal = sal + 2000 where empno = 7839;
   ~~~

   - 触发器与快照

     > 触发器：同步备份（同步：没有延迟）
     >
     > 快照：异步备份

   ​

   ​

