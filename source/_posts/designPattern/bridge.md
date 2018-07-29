---
title: java设计模式（十）桥接模式（Bridge）
date: 2017-09-13 17:14:07
categories: 
 - JAVA 
 - 设计模式
---

## 介绍

桥接模式就是把事物和其具体实现分开，使他们可以各自独立的变化。桥接的用意是：**将抽象化与实现化解耦，使得二者可以独立变化**，像我们常用的JDBC桥DriverManager一样，JDBC进行连接数据库的时候，在各个数据库之间进行切换，基本不需要动太多的代码，甚至丝毫不用动，原因就是JDBC提供统一接口，每个数据库提供各自的实现，用一个叫做数据库驱动的程序来桥接就行了。

类图：

![](bridge/dp1001.jpg)

<!-- more -->

接口：

~~~java
public interface Sourceable {  
    public void method();  
}  
~~~

实现类：

~~~java
public class SourceSub1 implements Sourceable {  
    @Override  
    public void method() {  
        System.out.println("this is the first sub!");  
    }  
}  
~~~

~~~java
public class SourceSub2 implements Sourceable {  
    @Override  
    public void method() {  
        System.out.println("this is the second sub!");  
    }  
}  
~~~

定义一个桥，持有Sourceable的一个实例：

~~~java
public abstract class Bridge {  
    private Sourceable source;  
  
    public void method(){  
        source.method();  
    }  
      
    public Sourceable getSource() {  
        return source;  
    }  
  
    public void setSource(Sourceable source) {  
        this.source = source;  
    }  
}  
~~~

~~~java
public class MyBridge extends Bridge {  
    public void method(){  
        getSource().method();  
    }  
}  
~~~

测试类：

~~~java
public class BridgeTest {      
    public static void main(String[] args) {         
        Bridge bridge = new MyBridge();  
          
        /*调用第一个对象*/  
        Sourceable source1 = new SourceSub1();  
        bridge.setSource(source1);  
        bridge.method();  
          
        /*调用第二个对象*/  
        Sourceable source2 = new SourceSub2();  
        bridge.setSource(source2);  
        bridge.method();  
    }  
}  
//输出：
//this is the first sub!
//this is the second sub!
~~~

这样，就通过对Bridge类的调用，实现了对接口Sourceable的实现类SourceSub1和SourceSub2的调用。接下来再画个图，JDBC连接的原理，有数据库学习基础的，一结合就都懂了。

![](bridge/dp1002.jpg)

## 扩展

### 1. 模式动机

- 对于有两个变化维度（即两个变化的原因）的系统，采用方案二来进行设计系统中类的个数更少，且系统扩展更为方便。桥接模式将继承关系转换为关联关系，从而降低了类与类之间的耦合，减少了代码编写量。
- 桥接模式(Bridge Pattern)：将抽象部分与它的实现部分分离，使它们都可以独立地变化。它是一种对象结构型模式，又称为柄体(Handle and Body)模式或接口(Interface)模式。

### 2. 模式结构

![](bridge/dp1003.png)

桥接模式包含如下角色：

- Abstraction：抽象类
- RefinedAbstraction：扩充抽象类
- Implementor：实现类接口
- ConcreteImplementor：具体实现类 

### 3. 模式分析

理解桥接模式，重点需要理解如何将抽象化(Abstraction)与实现化(Implementation)脱耦，使得二者可以独立地变化。

- 抽象化：抽象化就是忽略一些信息，把不同的实体当作同样的实体对待。在面向对象中，将对象的共同性质抽取出来形成类的过程即为抽象化的过程。
- 实现化：针对抽象化给出的具体实现，就是实现化，抽象化与实现化是一对互逆的概念，实现化产生的对象比抽象化更具体，是对抽象化事物的进一步具体化的产物。
- 脱耦：脱耦就是将抽象化和实现化之间的耦合解脱开，或者说是将它们之间的强关联改换成弱关联，将两个角色之间的继承关系改为关联关系。桥接模式中的所谓脱耦，就是指在一个软件系统的抽象化和实现化之间使用关联关系（组合或者聚合关系）而不是继承关系，从而使两者可以相对独立地变化，这就是桥接模式的用意。

实现类接口：

~~~java
public interface Implementor{
	public void operationImpl();
} 

~~~

抽象类：

~~~java
public abstract class Abstraction{
	protected Implementor impl;
	
	public void setImpl(Implementor impl){
		this.impl=impl;
	}
	
	public abstract void operation();
} 
~~~

扩充抽象类：

~~~java
public class RefinedAbstraction extends Abstraction{
	public void operation(){
		//代码
		impl.operationImpl();
		//代码
	}
} 
~~~

### 4. 模式适用场景

- 如果一个系统需要在构件的抽象化角色和具体化角色之间增加更多的灵活性，避免在两个层次之间建立静态的继承联系，通过桥接模式可以使它们在抽象层建立一个关联关系。
- 抽象化角色和实现化角色可以以继承的方式独立扩展而互不影响，在程序运行时可以动态将一个抽象化子类的对象和一个实现化子类的对象进行组合，即系统需要对抽象化角色和实现化角色进行动态耦合。
- 一个类存在两个独立变化的维度，且这两个维度都需要进行扩展。
- 虽然在系统中使用继承是没有问题的，但是由于抽象化角色和具体化角色需要独立变化，设计要求需要独立管理这两者。
- 对于那些不希望使用继承或因为多层次继承导致系统类的个数急剧增加的系统，桥接模式尤为适用。

### 5. 模式应用

- java虚拟机跨平台性

- JDBC驱动程序

  > 使用JDBC驱动程序的应用系统就是抽象角色，而所使用的数据库是实现角色。一个JDBC驱动程序可以动态地将一个特定类型的数据库与一个Java应用程序绑定在一起，从而实现抽象角色与实现角色的动态耦合。

### 6. 适配器模式与桥接模式联用

![](bridge/dp1004.png)

- 桥接模式和适配器模式用于设计的不同阶段，桥接模式用于系统的初步设计，对于存在两个独立变化维度的类可以将其分为抽象化和实现化两个角色，使它们可以分别进行变化；而在初步设计完成之后，当发现系统与已有类无法协同工作时，可以采用适配器模式。但有时候在设计初期也需要考虑适配器模式，特别是那些涉及到大量第三方应用接口的情况。

### 7. 模式优缺点

1. 优点
   - 分离抽象接口及其实现部分。
   - 桥接模式有时类似于多继承方案，但是多继承方案违背了类的单一职责原则（即一个类只有一个变化的原因），复用性比较差，而且多继承结构中类的个数非常庞大，桥接模式是比多继承方案更好的解决方法。
   - 桥接模式提高了系统的可扩充性，在两个变化维度中任意扩展一个维度，都不需要修改原有系统。
   - 实现细节对客户透明，可以对用户隐藏实现细节。
2. 缺点
   - 桥接模式的引入会增加系统的理解与设计难度，由于聚合关联关系建立在抽象层，要求开发者针对抽象进行设计与编程。
   - 桥接模式要求正确识别出系统中两个独立变化的维度，因此其使用范围具有一定的局限性。

