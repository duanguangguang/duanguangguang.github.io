---
title: oracle函数
date: 2017-08-27 17:11:33
categories: 
 - 数据库
tags:
 - Oracle
---

## Oracle之函数

### 一、数值函数

1. 四舍五入

   ~~~plsql
   SQL> ROUND(n[,m]);
   --省略m或者m等于0：取整
   --m>0：小数点后m位,保留m位小数
   --m<0：小数点前m位，从m位进行四舍五入

   SQL> select round(23.4),round(23.45,1),round(23.45,-1) from dual;
   --23, 23.5, 23
   ~~~
   <!--more -->

2. 取整函数

   ~~~plsql
   SQL> CEIL(n); --取最大值
   	 FLOOR(n); --取最小值

   SQL> select ceil(23.45),floor(23.45) from dual;
   --24, 23
   ~~~

3. 取绝对值

   ~~~plsql
   SQL> ABS(n);
   ~~~

4. 取余数

   ~~~plsql
   MOD(m,n);
   --m/n，如果 m和n中有一个值为null，则结果为null
   ~~~

5. m的n次幂

   ~~~plsql
   SQL> POWER(m,n);
   --m^n，m和n中有一个值为null，则结果为null
   ~~~

6. 平方根

   ~~~plsql
   SQL> SQRT(n);
   ~~~

7. 三角函数

   ~~~plsql
   SQL> SIN(n);ASIN(n); --正玄/反正玄
   	 COS(n);ACOS(n); --余玄/反余玄
   	 TAN(n);ATAN(n); --正切/反正切
   ~~~

### 二、字符函数

1. 大小写转换函数

   ~~~plsql
   SQL> UPPER(char); --小--大
   	 LOWER(char); --大--小
   	 INITCAP(char); --首字母大写

   SQL> select upper('abcd'),lower('ASD'),initcap('asd') from dual;
   --用途：注册用户名密码：不区分大小写，统一转换成大/小写存入数据库
   ~~~

2. 获取子字符串函数

   ~~~plsql
   SQL> SUBSTR(char[,m[,n]]);
   --char：源字符串
   --m：取子串开始的位置，m为负数表示从尾开始
   --n：截取的位数
   ~~~

3. 获取字符串长度函数

   ~~~plsql
   SQL> LENGTH(char)

   SQL> select length('abc ') from dual;--4
   ~~~

4. 字符串连接函数

   ~~~plsql
   SQL> CONCAT(char1,char2);--同||
   ~~~

5. 去除子串函数

   ~~~plsql
   --当只有一个参数时表示去除空格
   SQL> TRIM(c2 from c1); --从c1中去除c2，只能去除一个字符
   	 LTRIM(c1[,c2]); --从c1中去除c2，去除头部
   	 RTRIM(c1[,c2]); --从c1中去除c2，去除尾部

   SQL> select rtrim('abaa','a') from dual;--ab
   ~~~

6. 替换函数

   ~~~plsql
   SQL> REPLACE(char,s_string[,r_string]);
   --char：源字符串
   --s_string：源字符串中被替换的字符串
   --r_string：替换s_string的字符串，省略时表示空替换

   SQL> select replace('abcde','a','A') from dual;--Abcde
   	 select replace('abcde','a') from dual;--bcde
   ~~~

### 三、日期函数

1. 系统时间

   ~~~plsql
   SQL> SYSDATE;--默认格式是DD-MON-RR 27-8月-17
   ~~~

2. 日期操作

   ~~~plsql
   SQL> ADD_MONTHS(date,i);
    	 --i为正在月份上加，i为负在月份上减
   	 
   	 NEXT_DAY(date,char);
   	 select next_day(sysdate,'星期一') from dual;
   	 --返回下一个星期一是几号
   	 
   	 LAST_DAY(date);
   	 --用于返回日期所在月的最后一天是几号
   	 
   	 MONTHS_BETWEEN(date1,date2);
   	 --两日期之间间隔的月份
   	
   	 EXTRACT(date from datetime);
   	 select extract(year from sysdate) from dual;--month/day
   	 --返回所在的年/月/日
   	 select extract(hour from timestamp '2017-8-27 17:35:28') from dual;
   ~~~

### 四、转换函数

1. 日期转字符

   ~~~plsql
   SQL> TO_CHAR(date[,fmt[,params]]);
   --date：要转换的日期
   --fmt：转换的格式YY MM DD HH24 HH12 MI SS
   --params：日期的语言，英文的格式：YEAR MONTH DAY
   --默认格式：DD-MON-RR

   SQL> select to_char(sysdate,'YYYY-MM-DD HH24:MI:SS') from dual;
   ~~~

2. 字符转日期

   ~~~plsql
   SQL> TO_DATE(char[,fmt[,params]]);
   	 select to_date('2017-8-27','YYYY-MM-DD') from dual;
   --注：to_date按照系统默认格式显示成日期，可再结合to_char转换
   ~~~

3. 数字转字符

   ~~~plsql
   SQL> TO_CHAR(number[,fmt]);
   --9：显示数字并忽略前面的0
   --0：显示数字，位数不足用0补齐
   --.或D：显示小数点
   --,或G：显示千位符
   --$：显示美元符号
   --s：加正负号，前后都可以，不能同时加

   SQL> select to_char(12345.678,'$99,999.999') from dual;--$12,345.678
   ~~~

4. 字符转数字

   ~~~plsql
   SQL> TO_NUMBER(char[,fmt]);
   	 select to_number('$1,000','$9999') from dual;--1000
   ~~~

### 五、oracle在查询中使用函数

1. 使用字符函数

   ~~~plsql
   --根据身份证号得到生日
   SQL> select substr(cardid,7,8) from users;

   --将部门号“01”替换成“信息技术”
   SQL> select replace(deptno,'01','信息技术') from users;
   ~~~

2. 使用数值函数

   ~~~plsql
   SQL> select mod(age,10) from dual;
   ~~~

3. 使用日期函数

   ~~~plsql
   --取员工入职年份
   SQL> select extract(year from regdate) from users;

   --查询5月份入职的员工信息
   SQL> select * from users where extract(month from regdate) = 5;
   ~~~