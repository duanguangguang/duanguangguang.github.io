---
title: java设计模式（八）代理模式（Proxy）
date: 2017-09-12 19:07:09
categories: 
 - JAVA 
 - 设计模式
---

## 介绍

**代理模式**就是多一个代理类出来，替原对象进行一些操作。给某一个对象提供一个代理，并由代理对象控制对原对象的引用。代理模式的英文叫做Proxy或Surrogate，它是一种对象结构型模。

关系图：

![](proxy/dp801.jpg)

<!--more -->

接口：

~~~java
public interface Sourceable {  
    public void method();  
}  
~~~

被代理类：

~~~java
public class Source implements Sourceable {  
    @Override  
    public void method() {  
        System.out.println("the original method!");  
    }  
}  
~~~

代理类：

~~~java
public class Proxy implements Sourceable {  
    private Source source;  
    public Proxy(){  
        super();  
        this.source = new Source();  
    }  
    @Override  
    public void method() {  
        before();  
        source.method();  
        atfer();  
    }  
    private void atfer() {  
        System.out.println("after proxy!");  
    }  
    private void before() {  
        System.out.println("before proxy!");  
    }  
}  
~~~

测试类：

~~~java
public class ProxyTest {  
    public static void main(String[] args) {  
        Sourceable source = new Proxy();  
        source.method();  
    }  
  
}  
//输出：
//before proxy!
//the original method!
//after proxy!
~~~

**代理模式的应用场景：**

> 如果已有的方法在使用的时候需要对原有的方法进行改进，此时有两种办法：

- 修改原有的方法来适应。这样违反了**对扩展开放，对修改关闭**的原则。
- 就是采用一个代理类调用原有的方法，且对产生的结果进行控制。这种方法就是代理模式。

使用代理模式，可以将功能划分的更加清晰，有助于后期维护。

**装饰器模式和代理模式区别：**

- 这两个设计模式看起来很像。对装饰器模式来说，装饰者（decorator）和被装饰者（decoratee）都实现同一个 接口。对代理模式来说，代理类（proxy class）和真实处理的类（real class）都实现同一个接口。此外，不论我们使用哪一个模式，都可以很容易地在真实对象的方法前面或者后面加上自定义的方法。
- 然而，实际上，在装饰器模式和代理模式之间还是有很多差别的。装饰器模式关注于在一个对象上**动态的添加方法**，然而代理模式关注于**控制对对象的访问**。换句话说，用代理模式，代理类（proxy class）可以对它的客户隐藏一个对象的具体信息。因此，当使用代理模式的时候，我们常常**在一个代理类中创建一个对象的实例**。并且，当我们使用装饰器模 式的时候，我们通常的做法是**将原始对象作为一个参数传给装饰者的构造器**。
- 我们可以用另外一句话来总结这些差别：使用代理模式，代理和真实对象之间的的关系通常在编译时就已经确定了，而装饰者能够在运行时递归地被构造。

## 扩展

### 1. 模式动机

**作用：**在某些情况下，一个客户不想或者不能直接引用一个对象，此时可以通过一个称之为代理的第三者来实现间接引用。代理对象可以在客户端和目标对象之间起到中介的作用，并且可以通过代理对象去掉客户不能看到的内容和服务或者添加客户需要的额外服务。

**目的：**就是为其他对象提供一个代理以控制对某个对象的访问。代理类负责为委托类预处理消息，过滤消息并转发消息，以及进行消息被委托类执行后的后续处理。

### 2. 动态代理

- 动态代理是一种较为高级的代理模式，它的典型应用就是Spring AOP。

- 在传统的代理模式中，客户端通过Proxy调用Source类的method()方法，同时还在代理类中封装了其他方法(如before()和after())，可以处理一些其他问题。

- 如果按照这种方法使用代理模式，那么真实主题角色必须是事先已经存在的，并将其作为代理对象的内部成员属性。如果一个真实主题角色必须对应一个代理主题角色，这将导致系统中的类个数急剧增加，因此需要想办法减少系统中类的个数，此外，如何在事先不知道真实主题角色的情况下使用代理主题角色，这都是动态代理需要解决的问题。

- Java动态代理实现相关类位于java.lang.reflect包，主要涉及两个类：

  1. InvocationHandler接口

     > 它是代理实例的调用处理程序实现的接口，该接口中定义了如下方法：public Object invoke (Object proxy, Method method, Object[] args) throws Throwable。

     > invoke()方法中第一个参数proxy表示代理类，第二个参数method表示需要代理的方法，第三个参数args表示代理方法的参数数组。

  2. Proxy类

     > 该类即为动态代理类，该类最常用的方法为：public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) throws IllegalArgumentException。

     > newProxyInstance()方法用于根据传入的接口类型interfaces返回一个动态创建的代理类的实例，方法中第一个参数loader表示代理类的类加载器，第二个参数interfaces表示代理类实现的接口列表（与真实主题类的接口列表一致），第三个参数h表示所指派的调用处理程序类。

### 3. 代理模式涉及到的角色

- 抽象角色：声明真实对象和代理对象的共同接口。
- 代理角色：代理对象角色内部含有对真实对象的引用，从而可以操作真实对象，同时代理对象提供与真实对象相同的接口以便在任何时刻都能代替真实对象。同时，代理对象可以在执行真实对象操作时，附加其他的操作，相当于对真实对象进行封装。
- 真实角色：代理角色所代表的真实对象，是我们最终要引用的对象。

为了保持行为的一致性，代理类和委托类通常会实现相同的接口，所以在访问者看来两者没有丝毫的区别。通过代理类这中间一层，能有效控制对委托 类对象的直接访问，也可以很好地隐藏和保护委托类对象，同时也为实施不同控制策略预留了空间，从而在设计上获得了更大的灵活性。Java 动态代理机制以巧妙的方式近乎完美地实践了代理模式的设计理念。

## java动态代理

### 1. Proxy 

java.lang.reflect.Proxy：这是 Java 动态代理机制的主类，它提供了一组静态方法来为一组接口动态地生成代理类及其对象。

~~~java
// 方法 1: 该方法用于获取指定代理对象所关联的调用处理器
static InvocationHandler getInvocationHandler(Object proxy)
// 方法 2：该方法用于获取关联于指定类装载器和一组接口的动态代理类的类对象
static Class getProxyClass(ClassLoader loader, Class[] interfaces)
// 方法 3：该方法用于判断指定类对象是否是一个动态代理类
static boolean isProxyClass(Class cl)
// 方法 4：该方法用于为指定类装载器、一组接口及调用处理器生成动态代理类实例
static Object newProxyInstance(ClassLoader loader, Class[] interfaces,InvocationHandler h)
~~~

Proxy构造方法：

~~~java
// 由于 Proxy 内部从不直接调用构造函数，所以 private 类型意味着禁止任何调用
private Proxy() {}
// 由于 Proxy 内部从不直接调用构造函数，所以 protected 意味着只有子类可以调用
protected Proxy(InvocationHandler h) {this.h = h;}
~~~

Proxy 静态方法 newProxyInstance：

~~~java
public static Object newProxyInstance(ClassLoader loader,Class<?>[] interfaces,InvocationHandler h)
									throws IllegalArgumentException {
	
  	// 检查 h 不为空，否则抛异常
	if (h == null) {
		throw new NullPointerException();
	}
  
	// 获得与制定类装载器和一组接口相关的代理类类型对象
	Class cl = getProxyClass(loader, interfaces);
	// 通过反射获取构造函数对象并生成代理类实例
	try {
		Constructor cons = cl.getConstructor(constructorParams);
		return (Object) cons.newInstance(new Object[] { h });
	} catch (NoSuchMethodException e) { 
      	throw new InternalError(e.toString());
	} catch (IllegalAccessException e) { 
      	throw new InternalError(e.toString());
	} catch (InstantiationException e) { 
      	throw new InternalError(e.toString());
	} catch (InvocationTargetException e) { 
      	throw new InternalError(e.toString());
	}
}
~~~

**由此可见，动态代理真正的关键是在 getProxyClass 方法，该方法负责为一组接口动态地生成代理类类型对象。**

### 2. InvocationHandler 

java.lang.reflect.InvocationHandler：这是调用处理器接口，它自定义了一个 invoke 方法，用于集中处理在动态代理类对象上的方法调用，通常在该方法中实现对委托类的代理访问。

~~~java
// 该方法负责集中处理动态代理类上的所有方法调用。第一个参数既是代理类实例，第二个参数是被调用的方法对象
// 第三个方法是调用参数。调用处理器根据这三个参数进行预处理或分派到委托类实例上发射执行
Object invoke(Object proxy, Method method, Object[] args)
~~~

每次生成动态代理类对象时都需要指定一个实现了该接口的调用处理器对象（参见 Proxy 静态方法 4 的第三个参数）。

### 3. ClassLoader

java.lang.ClassLoader：这是类装载器类，负责将类的字节码装载到 Java 虚拟机（JVM）中并为其定义类对象，然后该类才能被使用。Proxy 静态方法生成动态代理类同样需要通过类装载器来进行装载才能使用，它与普通类的唯一区别就是其字节码是由 JVM 在运行时动态生成的而非预存在于任何个 .class 文件中。

每次生成动态代理类对象时都需要指定一个类装载器对象（参见 Proxy 静态方法 4 的第一个参数）

### 4. 代理机制及其特点

首先让我们来了解一下如何使用 Java 动态代理。**具体有如下四步骤：**

1. 通过实现 InvocationHandler 接口创建自己的调用处理器。
2. 通过为 Proxy 类指定 ClassLoader 对象和一组 interface 来创建动态代理类。
3. 通过反射机制获得动态代理类的构造函数，其唯一参数类型是调用处理器接口类型。
4. 通过构造函数创建动态代理类实例，构造时调用处理器对象作为参数被传入。

### 5. 动态代理对象创建过程

~~~java
// InvocationHandlerImpl 实现了 InvocationHandler 接口，并能实现方法调用从代理类到委托类的分派转发
// 其内部通常包含指向委托类实例的引用，用于真正执行分派转发过来的方法调用
InvocationHandler handler = new InvocationHandlerImpl(..);
// 通过 Proxy 为包括 Interface 接口在内的一组接口动态创建代理类的类对象
Class clazz = Proxy.getProxyClass(classLoader, new Class[] { Interface.class, ... });
// 通过反射从生成的类对象获得构造函数对象
Constructor constructor = clazz.getConstructor(new Class[] { InvocationHandler.class });
// 通过构造函数对象创建动态代理类实例
Interface Proxy = (Interface)constructor.newInstance(new Object[] { handler });

~~~

实际使用过程更加简单，因为 Proxy 的静态方法 newProxyInstance 已经为我们封装了步骤 2 到步骤 4 的过程，所以简化后的过程如下：

~~~java
// InvocationHandlerImpl 实现了 InvocationHandler 接口，并能实现方法调用从代理类到委托类的分派转发
InvocationHandler handler = new InvocationHandlerImpl(..);
// 通过 Proxy 直接创建动态代理类实例
Interface proxy = (Interface)Proxy.newProxyInstance( classLoader,new Class[] { Interface.class },
                                                     handler);
~~~

### 6. 一个动态代理的例子

接口：

~~~java
public interface Subject{
	public void request();
}
~~~

真实类：

~~~java
public class RealSubject implements Subject{
	public void request(){
		System.out.println("From real subject!");
	}
}
~~~

具体代理类：

~~~java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
public class DynamicSubject implements InvocationHandler{
  	private Object sub;
  	public DynamicSubject(Object obj){
		this.sub = obj;
	}
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable{
		System.out.println("before calling: " + method);
		method.invoke(sub, args);
		System.out.println(args == null);
		System.out.println("after calling: " + method);
		return null;
}
~~~

注：该代理类的内部属性是Object类型，实际使用的时候通过该类的构造方法传递进来一个对象。 此外，该类还实现了invoke方法，该方法中的method.invoke其实就是调用被代理对象的将要 执行的方法，方法参数是sub，表示该方法从属于sub，通过动态代理类，我们可以在执行真实对象的方法前后加入自己的一些额外方法。

测试类：

~~~java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
public class Client {
	public static void main(String[] args){
		RealSubject realSubject = new RealSubject();
		InvocationHandler handler = new DynamicSubject(realSubject);
		Class<?> classType = handler.getClass();

      	// 下面的代码一次性生成代理
		Subject subject = (Subject) Proxy.newProxyInstance(classType
							.getClassLoader(), realSubject.getClass().getInterfaces(),handler);
		subject.request();
		System.out.println(subject.getClass());
	}
}

~~~

