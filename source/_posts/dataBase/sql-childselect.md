---
title: Oracle高级查询之子查询
date: 2017-10-06 11:16:28
categories: 
 - 数据库
tags:
 - Oracle
---

### 介绍

查询工资比scott高的员工：

~~~sql
SQL> select * from emp where sal > (select sal from emp where ename = 'scott');
~~~

<!-- more -->

### 一、可以使用子查询的位置

1. select

   select 后的子查询只能是单行子查询，只有一条返回记录

   ~~~sql
   SQL> select empno,ename,sal,(select job from emp where empno = 7839) 第四列 from emp;
   ~~~

2. having

   ~~~sql
   SQL> select deptno,avg(sal) from emp group by deptno 
   	 having avg(sal) > (select max(sal) from emp where deptno = 30)
   ~~~

3. from

   ~~~sql
   SQL> select * from (select empno,ename,sal from emp);
   ~~~

4. 不可以使用子查询的位置：group by

### 二、子查询和多表查询

主查询与子查询不是同一张表。例：查询部门名称是sales的员工：

~~~sql
--子查询
SQL> select * from emp where deptno = (select deptno from dept where dname = 'sales');
--多表查询
SQL> select e.* from emp e, dept d
	 where e.deptno = d.deptno and d.name = 'sales';
~~~

sql优化：理论上采用多表查询好，只访问一次数据库

### 三、子查询的排序问题

top-n分析问题。例：排到员工表中工资最高的前三名：

~~~sql
SQL> select rownum,empno,ename,sal --rownum表示伪列
	 from (select * from emp order by sal desc)
	 where rownum <= 3;--行号只能用<,<=，不能使用>,>=(因为行号都是从1开始)
~~~

### 四、主查询和子查询的执行顺序

一般先子查询，后主查询，但相关子查询例外。例：找到员工表中薪水大于本部门平均水平的员工：

~~~sql
SQL> select empno,ename,sal,
	 (select avg(sal) from emp where deptno = e.deptno) avgsal
	 from emp e
	 where sal > (select avg(sal) from emp where deptno = e.deptno);
~~~

### 五、子查询中的null值

1. 单行子查询中null值问题

   查询条件一直为false，所以查询不到

2. 多行子查询中null值

   - a not in (1,2,null)
   - a != 1 and a != 2 and a != null
   - 子查询中增加不是空值的条件：coum  is not null

### 六、分页显示

分页查询显示员工信息：显示员工号、姓名、月薪：1.每页显示四条记录 2.显示第二页的员工信息（5-8）3.按照月薪降序排列

~~~sql
SQL> select r,empno,ename,sal
	 from (select rownum r,empno,ename,sal from(select rownum, --此时这个r理解为e2表的第一列（e1表的行号）
                                               empno,ename,sal from emp
                                               order by sal desc) e1
                                               where rownum <= 8) e2
	 where r >= 5;
~~~

### 七、sql执行计划

1. 执行计划：explain plan for sql语句;
2. 查看该执行计划：select * from table(dbms_xplan.display);
3. 判断标准：cost(%cpu)--消耗的系统资源。