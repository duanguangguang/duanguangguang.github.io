---
title: JAVA基础(五)反射机制
date: 2017-08-08 16:33:10
categories: 
 - JAVA
 - JavaSE
tags:
 - 反射
---

## JAVA基础之反射

java反射机制是在运行状态中，对于任意一个类（class文件），都能够知道这个类的所有属性和方法，对于任意一个对象，都能够调用它的任意一个方法和属性，这种动态获取的信息以及动态调用对象的方法的功能称为java的反射机制。可以理解为对类的解刨

在JDK中，主要由以下类来实现Java反射机制，这些类都位于java.lang.reflect包中

–Class类：代表一个类

–Field 类：代表类的成员变量（成员变量也称为类的属性）

–Method类：代表类的方法

–Constructor 类：代表类的构造方法

–Array类：提供了动态创建数组，以及访问数组的元素的静态方法

<!-- more -->

### 一、反射使用场景

1. 将名称配置在配置文件中

   如果想要对指定名称的字节码文件进行操作（扩展新功能），这时就使用到了反射技术

2. 反射就是拿到字节码的文件中的内容

   想要对一个类文件进行操作，只要获取到该类的字节码文件即可。Class类用来描述字节码文件，可以获取字节码文件中的所有内容，反射就是依靠该类来完成

### 二、获取Class对象的三种方式

1. Object类中的getClass方法

   这种方法比较麻烦，必须要明确具体的类，并创建了对象

   ~~~java
   Person p = new Person();//Person p1 = new Person();
   Class class = p.getClass();//Class class1 = p1.getClass();
   //class == class1
   ~~~

2. .class静态属性

   这样方法相对简单，任何数据类型都具备一个静态的属性.class来获取期对应的class对象，但还是要明确用到类中的静态成员，不够扩展

   ~~~java
   Class class = Person.class
   Class class1 = Person.class
   //class == class1
   ~~~

3. forName方法

   这种方法更为扩展，只要通过给定的类的字符串名称就可以获取该类

   ~~~java
   String className = "Person" //包名.类名
   Class class = Class.forName(className);
   ~~~

   该方法声明时需要抛classNotFoundException

### 三、获取Class中的构造函数

1. 无参

   ~~~java
   //获取类的字符串名称
   String name = "com.Person";
   //寻找该名称的类文件，并加载进内存，并产生class对象
   Class class = Class.forName(name);
   //产生此class对象的实例
   Object obj = class.newInstance();

   //若无空参数构造函数，则抛InstantiationException，初始化异常
   //若空参数为private，则抛IllegalAccessException，无效访问异常
   ~~~

2. 有参

   ~~~java
   com.Person p = new com.Person("小强",22);
   /*
   1.当要获取指定名称对应类中的所体现的对象时，
   2.而该对象初始化不使用空参数构造
   3.可以通过指定的构造函数进行初始化
   4.通过字节码文件对象获取构造函数
   5.方法是：
   */
   getConstructor(paramter types);
   getDeclaredConstructor(paramter types); //可以访问私有的
   ~~~

   例：

   ~~~java
   String name = "com.Person";
   Class class = Class.forName(name);
   //获取到指定的构造函数
   Constructor constructor = class.getConstructor(String.class,int.class);
   //通过该构造器对象的newInstance方法进行对象的初始化
   Object obj = constructor.newInstance("小强",22);
   ~~~

### 四、获取Class中的字段

~~~java
//方法
Field field = class.getField("age");//获取公有的
field = class.getDeclaredField("name");//可以获取私有的
//字段设置、获取值
field.setAccessible(true);//权限检查，暴力访问
//通过对象设、获取值
Object obj = class.newInstance();
field.set(obj,87);
Object o = field.get(obj);
~~~

### 五、获取Class中的方法

1. 获取Class中的公有函数

   ~~~java
   Method[] methods = class.getMethods();
   ~~~

2. 获取Class中本类中的函数，包括私有的

   ~~~java
   Method[] methods = class.getDeclaredMethods();
   ~~~

3. 取一个无参方法

   ~~~java
   //获取空参
   Method method = class.getMethod("show",null);
   //运行该方法
   Object obj = class.newInstance();
   method.invoke(obj,null);
   //带有给字段赋值的
   Constructor constructor = class.getConstructor(String.class,int.class);
   Object obj = constructor.newInstance("小明"，27);
   method.invoke(obj,null);
   ~~~

4. 取一个有参方法

   ~~~java
   Method method = class.getMethod("paramMethod",String.class,int.class);
   Object obj = class.newInstance();
   method.invoke(obj,"小强",39);
   ~~~

### 六、-Array类 

Integer.TYPE返回的是int，而Integer.class返回的是Integer类所对应的Class对象。java.lang.Array 类提供了动态创建和访问数组元素的各种静态方法

~~~java
//一维数组的简单创建，设值，取值
Object array = Array.newInstance(classType, 10);
Array.set(array, 5, "hello");
String str = (String)Array.get(array, 5);
~~~



### 七、反射练习

~~~java
//扩展PCI
Mainboard mb = new Mainboard();
File configFile = new File("pci.properties");
Properties prop = new Properties();
//将流对象加载进集合
FileInputStream fis = new FileInputStream(configFile);
prop.load(fis);

for(int x=0; x<prop.size(); x++){
  String pciName = prop.getProperty("pci"+(x+1));
  if(name != null){
    //用Class加载这个pci子类
    Class class = Class.forName(pciName);
    PCI p = (PCI)class.newInstance();
    mb.usePCI(p);
  }
}
fis.close();
~~~







