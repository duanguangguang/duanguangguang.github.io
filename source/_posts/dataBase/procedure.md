---
title: 存储过程和存储函数
date: 2017-08-27 22:52:55
categories: 
 - 数据库
tags:
 - Oracle
---

##  存储过程和存储函数

存储在数据库中供所有用户程序调用的子程序叫**存储过程**、**存储函数**。两者的相同点是：完成特点功能的程序。区别是：存储函数可以使用return语句返回值，存储过程不可以。存储过程的有点：

- 执行速度更快
- 允许模块化程序设计
- 提高系统安全性
- 减少网络流通量

系统存储过程存放在master数据库中，名称都以**sp_**开头或**xp_**开头，类似Java语言类库中的方法：

| 系统存储过程               | 说明                                   |
| -------------------- | ------------------------------------ |
| sp_databases         | 列出服务器上的所有数据库                         |
| sp_helpdb            | 报告有关指定数据库或所有数据库的信息                   |
| sp_renamedb          | 更改数据库的名称                             |
| sp_tables            | 返回当前环境下可查询的对象的列表                     |
| sp_columns           | 返回某个表列的信息                            |
| sp_help              | 查看某个表的所有信息                           |
| sp_helpconstraint    | 查看某个表的约束                             |
| sp_helpindex         | 查看某个表的索引                             |
| sp_stored_procedures | 列出当前环境中的所有存储过程                       |
| sp_password          | 添加或修改登录账户的密码                         |
| sp_helptext          | 显示默认值、未加密的存储过程、用户定义的存储过程、触发器或视图的实际文本 |

<!-- more -->

### 一、创建和使用存储过程

1. 用create procedure命令建立存储过程

~~~plsql
SQL> create [or replace] procedure 过程名[(参数列表)]
	 as
	 pl/sql子程序体;
~~~

- 第一个存储过程：打印hello world

  ~~~plsql
  SQL> create or replace procedure SAYHELLOWORLD
  	 as --oracle存储数据库对象默认采用大写，所以保存的过程名全大写
  	 --说明部分（包括声明光标等）
  	 begin
  		dbms_output.put_line('Hello World');
  	 end;
  	 /
  ~~~

- 调用该存储过程

  ~~~plsql
  --调用方式1
  SQL> exec SAYHELLOWORLD();
  ----调用方式2
  SQL> begin
  		SAYHELLOWORLD();
  	 end;
  	 /
  ~~~

2. 带参数的存储过程

   - 例：为指定的员工涨100块钱工资，并打印涨前和涨后的工资

     ~~~plsql
     SQL> create or replace procedure RAISESALARY(eno in number) --输入参数
     	 as 
     	 --定义一个变量保存涨前的薪水
     	 psal emp.sal % type;
     	 begin
     		--得到员工涨前的薪水
     		select sal into psal from emp where empno = eno;
     		--涨薪水操作
     		update emp set sal = sal + 100 where empno = eno;
     		--可以commit和rollback
     		--一般不在存储过程或存储函数中commit或rollback，因为这样不能保证调用者在同一个事务中
     		dbms_output.put_line('涨前：'|| psal || '涨后：' || (psal+100));
     	 end;
     	 /
     ~~~

   - 调用该存储过程

     ~~~plsql
     SQL> begin
     		RAISESALARY(7839);
     		RAISESALARY(7840);
     		commit;
     	 end;
     	 /
     ~~~

3. 调用存储过程

   - 不推荐远程调试（指oracle数据库与图形化工具不在同一机器上），这样远程调试需要指定服务器的IP地址，还需要工具-首选项-调试器-端口

   - 推荐把图形化工具拷贝到虚拟机所在的服务器的IP地址上

   - 授权

     ~~~plsql
     SQL> / as sysdba;--不用写用户名和密码，采用主机认证
     	 grant debug connect session,debug any procedure to scott;
     ~~~

### 二、存储函数

**函数(Function)**命名的存储程序，可带参数，并返回一计算值。存储函数和存储过程的结构类似，但必须有一个return子句用于返回函数值

- 创建存储函数的语法

  ~~~plsql
  SQL> create [or replace] function 函数名[(参数列表)]
  	 return 函数值类型
  	 as
  	 pl/sql子程序体;
  ~~~

- 例：查询某个员工的年收入

  ~~~plsql
  SQL> create or replace function QUERYEMPINCOME(eno in number)
  	 return number
  	 as
  	 --定义员工的薪水和奖金变量
  	 psal emp.sal % type;
  	 pcomm emp.comm % type;
  	 begin
  		--得到该员工的月薪和奖金
  		select sal,comm into psal,pcomm from emp where empno = eno;
  		return psal*12+nvl(pcomm,0);
  	 end;
  	 /
  ~~~

### 三、out参数

存储过程和存储函数都可以有多个out参数，存储过程可以通过out参数来实现返回值。原则：如果只有一个返回值，则使用存储 函数，否则使用存储过程。例：查询某个员工姓名、月薪和职位：

~~~plsql
SQL> create or replace procedure QUERYEMPININFO(eno in number,
                                                pename out varchar2,
	                                            psal out number,
                                                pjob out varchar2)
	 
	 as
	 begin
		select ename,sal,empjob into pename,psal,pjob from emp where empno = eno;
	 end;
	 /
~~~

分析：

1. 查询某个员工的所有信息--out参数太多？

2. 查询某个部门中所有员工的所有信息--out参数中返回集合？

3. > 如何解决：在out参数中使用光标

### 四、out参数使用光标

查询某个部门中所有员工的所有信息：

~~~plsql
--包头：只负责声明ref---引用，
SQL> create or replace package mypackage
	 as
	 type empcursor is ref cursor;--光标的类型，使用type自定义的
	 --ref---引用，引用光标的类型作为empcursor的类型，即empcursor为光标类型
	 procedure QUERYEMPLIST(dno in number, emplist out empcursor);
	 end mypackage;
--包体：需要实现包头中声明的所有方法，包括存储过程
SQL> create or replace package body mypackage
	 as
	 	procedure QUERYEMPLIST(dno in number, emplist out empcursor);
	 as
	 begin
	 	open emplist for select * from emp where deptno = dno;--for关键字
		end QUERYEMPLIST;
	 end mypackage;
--可以查看程序包的结构
SQL> desc mypackage
~~~

在应用中实现

~~~java
//在应用中访问包中的存储过程
package.cursor//需要带包名
//应用程序实现
rs = ((OracleCallableStatement)call).getCursor(2);//需要强制转换
~~~

### 五、java访问存储过程

数据库工具类

~~~java
/**
 * 接口：java.sql.CallableStatement
 * 该语法允许对所有RDBMS使用标准方式调用存储过程
 * @author guangguang_duan
 *
 */

//创建数据库工具类
public class JDBCUtils {
	private static String driver = "oracle.jdbc.OracleDriver";
	private static String url = "jdbc:oracle:thin:@127.0.0.1:1521:orcl";
	private static String user = "scott";
	private static String password = "tiger";
	//注册数据库的驱动
	static{
		try{
			Class.forName(driver);//使用反射机制注册
			//DriverManager.registerDriver(driver);
		}catch(ClassNotFoundException e){
			throw new ExceptionInInitializerError(e);
		}
	}
	//获取数据库连接
	public static Connection getConnection(){
		try{
			return DriverManager.getConnection(url,user,password);
		}catch(SQLException e){
			e.printStackTrace();
		}
		return null;
	}
	//释放数据库方法
	public static void release(Connection conn, Statement st, ResultSet rs){
		if(rs != null){
			try{
				rs.close();
			}catch(SQLException e){
				e.printStackTrace();
			}finally{
				rs = null;//目的是这个对象会迅速成为java垃圾回收的对象
			}
		}
		if(st != null){
			try{
				st.close();
			}catch(SQLException e){
				e.printStackTrace();
			}finally{
				st = null;
			}
		}
		if(conn != null){
			try{
				conn.close();
			}catch(SQLException e){
				e.printStackTrace();
			}finally{
				conn = null;
			}
		}
	}
}
~~~

访问存储过程

~~~java
import org.junit.Test;
public class TestProcedure {
	@Test
	public void tProcedure(){
		String sql = "{call QUERYEMPINFO(?,?,?,?)}";
		Connection conn = null;
		CallableStatement call = null;
		try{
			conn = JDBCUtils.getConnection();
			call = conn.prepareCall(sql);
			//对于in参数，赋值
			call.setInt(1, 7839);
			//对于out参数，申明
			call.registerOutParameter(2, OracleTypes.VARCHAR);//???
			call.registerOutParameter(3, OracleTypes.NUMBER);
			call.registerOutParameter(4, OracleTypes.VARCHAR);
			//执行调用
			call.execute();
			//取结果
			String name = call.getString(2);
			double sal = call.getDouble(3);
			String job = call.getString(4);
			System.out.println(name +"\t"+ sal +"\t"+job);
		}catch(Exception e){
			e.printStackTrace();	
		}finally{
			JDBCUtils.release(conn, call, null);
		}
	}
}
~~~

### 六、java访问存储函数

访问存储函数

~~~java
import org.junit.Test;
public class TestFunction {
	@Test
	public void tProcedure(){
		//第一个?表示out参数，第二个?表示in参数
		String sql = "{? = call QUERYEMPINCOME(?)}";
		
		/**
		 * 同存储过程
		 */
	}
}
~~~

