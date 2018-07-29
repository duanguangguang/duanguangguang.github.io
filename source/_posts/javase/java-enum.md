---
title: JAVA基础（六）枚举类
date: 2017-08-08 19:21:08
categories: 
 - JAVA
 - JavaSE
tags:
 - JavaSE
---

## java基础之枚举类

枚举类型是JDK5.0的新特征，常见使用于switch语句

### 一、枚举类的定义

~~~java
public enum Color{
RED,BLUE,BLACK,YELLOW,GREEN
}
~~~

<!-- more -->

显然，enum很像特殊的class，实际上enum声明定义的类型就是一个类。 而这些类都是类库中Enum类的子类，代码编译之后发现，编译器将 enum类型单独编译成了一个字节码文件：Color.class

~~~java
final enum hr.test.Color {
// 所有的枚举值都是类静态常量
public static final enum hr.test.Color RED;
public static final enum hr.test.Color BLUE;
public static final enum hr.test.Color BLACK;
public static final enum hr.test.Color YELLOW;
public static final enum hr.test.Color GREEN;
private static final synthetic hr.test.Color［］ ENUM$VALUES;
}
~~~

Color枚举类就是class，而且是一个不可以被继承的final类。其枚举值（RED,BLUE...）都是Color类型的类静态常量， 我们可以通过下面的方式来得到Color枚举类的一个实例：Color c=Color.RED

### 二、枚举类的构造器

1. 构造器只是在构造枚举值的时候被调用

   ~~~java
   enum Color{
   RED（255，0，0），BLUE（0，0，255），BLACK（0，0，0），YELLOW（255，255，0），GREEN（0，255，0）;
   //构造枚举值，比如RED（255，0，0）
   private Color（int rv，int gv，int bv）{
   this.redValue=rv;
   this.greenValue=gv;
   this.blueValue=bv;
   }
   public String toString（）{ //覆盖了父类Enum的toString（）
   return super.toString（）+“（”+redValue+“，”+greenValue+“，”+blueValue+“）”;
   }
   private int redValue; //自定义数据域，private为了封装。
   private int greenValue;
   private int blueValue;
   }
   ~~~

2. 构造器只能私有private，绝对不允许有public构造器。 这样可以保证外部代码无法新构造枚举类的实例

### 三、枚举类的方法

~~~java
//ordinal方法： 返回枚举值在枚举类种的顺序。这个顺序根据枚举值声明的顺序而定。
Color.RED.ordinal（）; //返回结果：0
Color.BLUE.ordinal（）; //返回结果：1

//compareTo方法： Enum实现了java.lang.Comparable接口，因此可以比较象与指定对象的顺序。Enum中的compareTo返回的是两个枚举值的顺 序之差。当然，前提是两个枚举值必须属于同一个枚举类，否则会抛出ClassCastException异常
Color.RED.compareTo（Color.BLUE）; //返回结果 -1

//values方法： 静态方法，返回一个包含全部枚举值的数组。
Color［］ colors=Color.values（）;
for（Color c:colors）{
System.out.print（c+“,”）;//返回结果：RED,BLUE,BLACK,YELLOW,GREEN...
}

//toString方法： 返回枚举常量的名称。
Color c=Color.RED;
System.out.println（c）;//返回结果： RED

//valueOf方法： 这个方法和toString方法是相对应的，返回带指定名称的指定枚举类型的枚举常量。
Color.valueOf（“BLUE”）; //返回结果： Color.BLUE

//equals方法： 比较两个枚举类对象的引用。
//JDK源代码：
public final boolean equals（Object other） {
return this==other;
}
~~~





