---
title: java设计模式（三）单例模式（Singleton）
date: 2017-09-09 15:14:16
categories: 
 - JAVA 
 - 设计模式
---

## 介绍

单例对象（Singleton）是一种常用的设计模式。在Java应用中，单例对象能保证在一个JVM中，该对象只有一个实例存在，而且自行实例化并向整个系统提供这个实例。这样的模式有几个好处：

1. 某些类创建比较频繁，对于一些大型的对象，这是一笔很大的系统开销。
2. 省去了new操作符，降低了系统内存的使用频率，减轻GC压力。
3. 有些类如交易所的核心交易引擎，控制着交易流程，如果该类可以创建多个的话，系统完全乱了。所以只有使用单例模式，才能保证核心交易服务器独立控制整个流程。

在单例模式的实现过程中，需要注意如下三点：

1. 单例类的构造函数为私有
2. 提供一个自身的静态私有成员变量
3. 提供一个公有的静态工厂方法

<!--more -->

## 非线程安全单例

~~~java
public class Singleton {  
    /* 持有私有静态实例，防止被引用，此处赋值为null，目的是实现延迟加载 */  
    private static Singleton instance = null;  
  
    /* 私有构造方法，防止被实例化 */  
    private Singleton() {  
    }  
  
    /* 静态工程方法，创建实例 */  
    public static Singleton getInstance() {  
        if (instance == null) {  
            instance = new Singleton();  
        }  
        return instance;  
    }  
  
    /* 如果该对象被用于序列化，可以保证对象在序列化前后保持一致 */  
    public Object readResolve() {  
        return instance;  
    }  
}  
~~~

这个类可以满足基本要求，但是，像这样毫无线程安全保护的类，如果我们把它放入多线程的环境下，肯定就会出现问题了。如何解决？

首先会想到对getInstance方法加synchronized关键字，如下：

~~~java
public static synchronized Singleton getInstance() {  
        if (instance == null) {  
            instance = new Singleton();  
        }  
        return instance;  
    }  
~~~

但是，synchronized关键字锁住的是这个对象，这样的用法，在性能上会有所下降，因为每次调用getInstance()，都要对对象上锁，事实上，只有在第一次创建对象的时候需要加锁，之后就不需要了，所以，这个地方需要改进：

~~~java
public static Singleton getInstance() {  
        if (instance == null) {  
            synchronized (instance) {  
                if (instance == null) {  
                    instance = new Singleton();  
                }  
            }  
        }  
        return instance;  
    }  
~~~

这样似乎解决了之前提到的问题，将synchronized关键字加在了内部，也就是说当调用的时候是不需要加锁的，只有在instance为null，并创建对象的时候才需要加锁，性能有一定的提升。但是，这样的情况，还是有可能有问题的，看下面的情况：在Java指令中创建对象和赋值操作是分开进行的，也就是说instance = new Singleton();语句是分两步执行的。但是JVM并不保证这两个操作的先后顺序，也就是说有可能JVM会为新的Singleton实例分配空间，然后直接赋值给instance成员，然后再去初始化这个Singleton实例。这样就可能出错了，我们以A、B两个线程为例：

1. A、B线程同时进入了第一个if判断
2. A首先进入synchronized块，由于instance为null，所以它执行instance = new Singleton()
3. 由于JVM内部的优化机制，JVM先画出了一些分配给Singleton实例的空白内存，并赋值给instance成员（注意此时JVM没有开始初始化这个实例），然后A离开了synchronized块
4. B进入synchronized块，由于instance此时不是null，因此它马上离开了synchronized块并将结果返回给调用该方法的程序
5. 此时B线程打算使用Singleton实例，却发现它没有被初始化，于是错误发生了

所以程序还是有可能发生错误，其实程序在运行过程是很复杂的，从这点我们就可以看出，尤其是在写多线程环境下的程序更有难度，有挑战性。我们对该程序做进一步优化：

~~~java
private static class SingletonFactory{           
        private static Singleton instance = new Singleton();           
    }           
    public static Singleton getInstance(){           
        return SingletonFactory.instance;           
    }   
~~~

实际情况是，单例模式使用内部类来维护单例的实现，JVM内部的机制能够保证当一个类被加载的时候，这个类的加载过程是线程互斥的。这样当我们第一次调用getInstance的时候，JVM能够帮我们保证instance只被创建一次，并且会保证把赋值给instance的内存初始化完毕，这样我们就不用担心上面的问题。同时该方法也只会在第一次调用的时候使用互斥机制，这样就解决了低性能问题：

~~~java
public class Singleton {  
    /* 私有构造方法，防止被实例化 */  
    private Singleton() {  
    }  
  
    /* 此处使用一个内部类来维护单例 */  
    private static class SingletonFactory {  
        private static Singleton instance = new Singleton();  
    }  
  
    /* 获取实例 */  
    public static Singleton getInstance() {  
        return SingletonFactory.instance;  
    }  
  
    /* 如果该对象被用于序列化，可以保证对象在序列化前后保持一致 */  
    public Object readResolve() {  
        return getInstance();  
    }  
}  
~~~

如果在构造函数中抛出异常，实例将永远得不到创建，也会出错。所以说，十分完美的东西是没有的，我们只能根据实际情况，选择最适合自己应用场景的实现方法。也有人这样实现：因为我们只需要在创建类的时候进行同步，所以只要将创建和getInstance()分开，单独为创建加synchronized关键字，也是可以的：

~~~java
public class SingletonTest {  
    private static SingletonTest instance = null;  
  
    private SingletonTest() {  
    }  
  
    private static synchronized void syncInit() {  
        if (instance == null) {  
            instance = new SingletonTest();  
        }  
    }  
  
    public static SingletonTest getInstance() {  
        if (instance == null) {  
            syncInit();  
        }  
        return instance;  
    }  
}  
~~~

考虑性能的话，整个程序只需创建一次实例，所以性能也不会有什么影响

总结：synchronized关键字锁定的是对象，在用的时候，一定要在恰当的地方使用（注意需要使用锁的对象和过程，可能有的时候并不是整个对象及整个过程都需要锁）

## 影子实例

采用**影子实例**的办法为单例对象的属性同步更新：

~~~java
public class SingletonTest {  
    private static SingletonTest instance = null;  
    private Vector properties = null;  
  
    public Vector getProperties() {  
        return properties;  
    }  
  
    private SingletonTest() {  
    }  
  
    private static synchronized void syncInit() {  
        if (instance == null) {  
            instance = new SingletonTest();  
        }  
    }  
  
    public static SingletonTest getInstance() {  
        if (instance == null) {  
            syncInit();  
        }  
        return instance;  
    }  
  
    public void updateProperties() {  
        SingletonTest shadow = new SingletonTest();  
        properties = shadow.getProperties();  
    }  
}  
~~~

## 静态类与单例类

采用类的静态方法，实现单例模式的效果，也是可行的，二者有什么不同？

- 首先，静态类不能实现接口。（从类的角度说是可以的，但是那样就破坏了静态了。因为接口中不允许有static修饰的方法，所以即使实现了也是非静态的）
- 其次，单例可以被延迟初始化，静态类一般在第一次加载时初始化。之所以延迟加载，是因为有些类比较庞大，所以延迟加载有助于提升性能
- 再次，单例类可以被继承，他的方法可以被覆写。但是静态类内部方法都是static，无法被覆写
- 最后一点，单例类比较灵活，毕竟从实现上只是一个普通的Java类，只要满足单例的基本需求，你可以在里面随心所欲的实现一些其它功能，但是静态类不行。从上面这些概括中，基本可以看出二者的区别，但是，从另一方面讲，我们上面最后实现的那个单例模式，内部就是用一个静态类来实现的，所以，二者有很大的关联，只是我们考虑问题的层面不同罢了。两种思想的结合，才能造就出完美的解决方案，就像HashMap采用数组+链表来实现一样

## 懒汉模式

~~~java
public class Singleton{
	private static Singleton instance=null;  //静态私有成员变量
	//私有构造函数
	private Singleton(){	
	}
	
    //静态公有工厂方法，返回唯一实例
	public static Singleton getInstance(){
		if(instance==null)
		    instance=new Singleton();	
		return instance;
	}
}
~~~

## 饿汉模式

~~~java
public class Singleton{
	private static Singleton instance=new Singleton() ;  //静态私有成员变量
	//私有构造函数
	private Singleton(){	
	}
	
    //静态公有工厂方法，返回唯一实例
	public static Singleton getInstance(){
		return instance;
	}
}
~~~

## 枚举类型模式

~~~java
public enum Sinleton {
    INSTANCE;
    public void doSomething() {
      	System.out.println("enum Singleton");
    }
}
~~~

测试类：

~~~java
public class Test{
   public static void main(String[] args){
      Sinleton.INSTANCE.doSomething();
   }
}
~~~

## 单例模式应用

~~~java
public class Runtime {
    private static Runtime currentRuntime = new Runtime();
    private Runtime() {
      
    } 
  	public static Runtime getRuntime() { 
		return currentRuntime;
    }
} 
~~~

适用场景

- 需要频繁实例化然后销毁的对象。
- 创建对象时耗时过多或者耗资源过多，但又经常用到的对象。
-  有状态的工具类对象。
-  频繁访问数据库或文件的对象。
-  以及其他我没用过的所有要求只有一个对象的场景。

## 单例模式的优缺点

1. 优点
   - 在内存中只有一个对象，节省内存空间。
   -  避免频繁的创建销毁对象，可以提高性能。
   -  避免对共享资源的多重占用。
   -  可以全局访问。


2. 缺点
   - 由于单例模式中没有抽象层，因此单例类的扩展有很大的困难。
   - 单例类的职责过重，在一定程度上违背了“单一职责原则”。因为单例类既充当了工厂角色，提供了工厂方法，同时又充当了产品角色，包含一些业务方法，将产品的创建和产品的本身的功能融合到一起。
   - 滥用单例将带来一些负面问题，如为了节省资源将数据库连接池对象设计为单例类，可能会导致共享连接池对象的程序过多而出现连接池溢出；现在很多面向对象语言(如Java、C#)的运行环境都提供了自动垃圾回收的技术，因此，如果实例化的对象长时间不被利用，系统会认为它是垃圾，会自动销毁并回收资源，下次利用时又将重新实例化，这将导致对象状态的丢失。
3. 注意事项
   - 只能使用单例类提供的方法得到单例对象，不要使用反射，否则将会实例化一个新对象。
   - 不要做断开单例类对象与类中静态引用的危险操作。
   - 多线程使用单例使用共享资源时，注意线程安全问题。

## 思考

1. 单例模式的对象长时间不用会被jvm垃圾收集器收集吗

   看到不少资料中说：如果一个单例对象在内存中长久不用，会被jvm认为是一个垃圾，在执行垃圾收集的时候会被清理掉。对此这个说法，网上一个观点是：在hotspot虚拟机1.6版本中，除非人为地断开单例中静态引用到单例对象的联接，否则jvm垃圾收集器是不会回收单例对象的。

2. 在一个jvm中会出现多个单例吗

   在分布式系统、多个类加载器、以及序列化的的情况下，会产生多个单例，这一点是无庸置疑的。那么在同一个jvm中，会不会产生单例呢？使用单例提供的getInstance()方法只能得到同一个单例，除非是使用反射方式，将会得到新的单例。代码如下

   ~~~java
   Class c = Class.forName(Singleton.class.getName());
   Constructor ct = c.getDeclaredConstructor();
   ct.setAccessible(true);
   Singleton singleton = (Singleton)ct.newInstance();
   ~~~

   这样，每次运行都会产生新的单例对象。所以运用单例模式时，一定注意不要使用反射产生新的单例对象。

3. 懒汉式单例线程安全吗

   主要是网上的一些说法，懒汉式的单例模式是线程不安全的，即使是在实例化对象的方法上加synchronized关键字，也依然是危险的。但事实加synchronized关键字修饰后，虽然对性能有部分影响，但是却是线程安全的，并不会产生实例化多个对象的情况。

4. 单例模式只有饿汉式和懒汉式两种吗

   饿汉式单例和懒汉式单例只是两种比较主流和常用的单例模式方法，从理论上讲，任何可以实现一个类只有一个实例的设计模式，都可以称为单例模式。

5. 单例类可以被继承吗

   饿汉式单例和懒汉式单例由于构造方法是private的，所以他们都是不可继承的，但是其他很多单例模式是可以继承的，例如登记式单例。

6. 饿汉式单例好还是懒汉式单例好

   在java中，饿汉式单例要优于懒汉式单例。C++中则一般使用懒汉式单例。