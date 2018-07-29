---
title: JAVA基础(二)常用类
date: 2017-07-27 09:59:36
categories: 
 - JAVA
 - JavaSE
tags:
 - JavaSE
---

## JAVA基础之常用类

之前很容易忽视的几个java常用类，在jdk1.1之后，应使用Calendar类实现日期和时间字段之间转换，使用DateFormat类来格式化和解析日期字符串

<!-- more -->

### Object

所有类的基类，是不断抽取而来的，具备所有对象都具备的内容

1. equals

   本质上还是调用 “==”，比较对象的地址值

   重写该方法，进行向下类型转型，需要注意转换异常，进行健壮性判断

2. hashCode

   hashCode方法的常规协定，该协定声明相等的对象必须具有相等的哈希码

### System类

不能被实例化，都是static方法

~~~java
long currentTimeMillis();  //获取当前时间的毫秒值
Properties getProperties();//获取当前系统属性
~~~

获取系统的属性信息，并存储到Properties集合中，该集合中存储的都是String类型的键和值，最好使用他自己的存储和取出的方法操作元素

### Runtime 类

没有构造方法，说明该类不可以创建对象，又发现还有一个非静态的方法，说明该类应该提供静态的返回该类对象的方法，而且只有一个，说明Runtime类使用了单例设计模式（保证运行时java对象唯一性）

~~~java
Runtime r = Runtime.getRuntime();//运行时对象产生
r.exec("nodepad.exe"); //执行execute
r.exec("notepad.exe c: \\Runtime.java");//用程序解析文件

Process p = r.exe("nodepad.exe");
Thread.sleep(5000);
p.destroy(); //销毁nodepad.exe这个进程
~~~



### DateFormat类

不能创建实例对象

1. format方法

   将日期对象转换成日期格式字符串

   ~~~java
   String myString = DateFormat.getDateInstance().format(myDate); //日期
   String myString = DateFormat.getDateTimeInstance().format(myDate); //时间
   //具有默认风格FULL,LONG等
   DateFormat dateFormat = DateFormat.getDateInstance(DateFormat.FULL);
   String myString = dateFormat.format(myDate);
   //自定义风格
   SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy:mm:dd");
   //SimpleDateFormat是DateFormat的子类，可以创建对象
   String myString = dateFormat.format(myDate);
   ~~~

   ​

2. parse方法

   将日期格式的字符串转换成日期对象

   ~~~java
   String str_date = "2017-07-28";
   DateFormat dateFormat = DateFormat.getDateInstance();//默认的，可以解析--
   Date date = dateFormat.parse(str_date);
   //自定义风格
   str_date = "2017---07---28";
   dateFormat = new SimpleDateFormat("yyyy---MM---dd");
   //年月日
   str_date = "2017年7月28号";
   dateFormat = DateFormat.getDateInstance(DateFormat.LONG);
   ~~~

   ​

### Calendar类

日历类，替代Date类

~~~java
Calendar c = Calendar.getInstance();
int year = c.get(Calendar.YEAR);
int month = c.get(Calendar.MONTH)+1; //注意月份要+1
int day = c.get(Calendar.DAY_OF_MONTH);
int week = getWeek(c.get(Calendar.DAY_OF_WEEK));
//设置日期
c.set(2017,7,28);
//偏移
c.add(Calendar.YEAR,2);
c.add(Calendar.YEAR,-2);
~~~



### 包装类

Java为基本类型提供包装类，这使得任何接受对象的操作也可以用来操作基本类型，直接将简单类型的变量表示为一个类，在执行变量类型的相互转换时，我们会大量使用这些包装类。java是一种面向对象语言，java中的类把方法与数据连接在一起，并构成了自包含式的处理单元。但在java中不能定义基本类型(primitive type)，为了能将基本类型视为对象来处理，并能连接相关的方法，java为每个基本类型都提供了包装类，这样，我们便可以把这些基本类型转化为对象来处理了。这些包装类有：Boolean，Byte，Short，Character，Integer，Long，Float等

java是可以直接处理基本类型的，但是在有些情况下我们需要将其作为对象来处理，这时就需要将其转化为包装类了。所有的包装类(Wrapper Class)都有共同的方法，他们是：

1. 带有基本值参数并创建包装类对象的构造函数。如可以利用Integer包装类创建对象

   ```java
   Integer obj = new Integer(145);
   ```

2. 带有字符串参数并创建包装类对象的构造函数

   ~~~java
   new Integer("45");
   ~~~

3. 生成字符串表示法的toString()方法

   ~~~java
   obj.toString();
   ~~~

4. 对同一个类的两个对象进行比较的equals()方法

   ~~~java
   obj1.eauqls(obj2);
   ~~~

5. 生成哈稀表代码的hashCode方法

   ~~~java
   obj.hasCode();
   ~~~

6. 将字符串转换为基本值的 parseType方法

   ~~~java
   Integer.parseInt(args[0]);
   ~~~

   注意：Character没有parse方法，但有forDigit方法

7. 可生成对象基本值的typeValue方法

   ~~~java
   obj.intValue();
   ~~~

包装类对象比较大小使用compareTo方法（1   0   -1）

进制转换

1. 十进制转其他进制

   ~~~java
   Integer.toBinaryString(2); //二进制
   Integer.toOctalString(8);  //八进制
   Integer.toHexString(16);   //十六进制
   Integer.toString(100,4);   //四进制
   ~~~

2. 其他转十进制

   ~~~java
   Integer.parseInt("110",2); //二进制转十进制
   Integer.parseInt("3c",16); //十六进制转十进制
   ~~~

#### 包装类的自动装箱拆箱

1. 装箱：基本数据类型赋值给引用数据类型叫装箱

   自动装箱的过程：每当需要一种类型的对象时，这种基本类型就自动地封装到与它相同类型的包装中

~~~java
Integer i = 4；
//Integer i = new Integer(4);
~~~

2. 拆箱：当基本数据类型和引用数据类型做运算时

   自动拆箱的过程：每当需要一个值时，被装箱对象中的值就被自动地提取出来，没必要再去调用intValue()和doubleValue()方法

~~~
Integer i = 4;
int k = i + 6;
~~~

3. 当Integer i = null时，拆箱时会调用intValue方法会产生异常，需要进行健壮性判断

> Integer的自动装拆箱注意细节

~~~java
Integer a = 100;
Integer b = 100;
System.out.println(a==b); //true
~~~

比较的时候，还是比较对象的reference，但是自动装箱时，java在编译的时候 Integer a = 100；被翻译成Integer a = Integer.valueOf(100)。结果为true的原因就是这个valueOf方法

~~~java 
public static Integer valueOf(int i) {
	final int offset = 128;
	if (i >= -128 && i <= 127) { // must cache
		return IntegerCache.cache[i + offset];
	}
	return new Integer(i);
}
	
private static class IntegerCache {
	private IntegerCache(){
      
	}
	static final Integer cache[] = new Integer[-(-128) + 127 + 1];//将cache[]变成静态
	static {//初始化一次，在对象间共享，也就是不同的对象共享同一个static数据
		for(int i = 0; i < cache.length; i++)
			cache = new Integer(i - 128);//初始化cache[i]
	}
} 
~~~

根据上面的jdk源码，java为了提高效率，IntegerCache类中有一个数组缓存 了值从-128到127的Integer对象。当我们调用Integer.valueOf（int i）的时候，如果i的值是>=-128且<=127时，会直接从这个缓存中返回一个对象，否则就new一个Integer对象。 

