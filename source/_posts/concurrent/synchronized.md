---
title: java内置锁synchronized
date: 2018-12-26 20:21:31
tags:
 - 并发包
 - 锁
 - 面试
categories: 
 - 并发包
---

### 介绍

在多线程编程中，线程安全问题是一个最为关键的问题，其核心概念就在于正确性，即当多个线程访问某一共享、可变数据时，始终都不会导致数据破坏以及其他不该出现的结果。而所有的并发模式在解决这个问题时，采用的方案都是序列化访问临界资源 。在 Java 中，提供了两种方式来实现同步互斥访问：synchronized 和 Lock。

<!-- more -->

线程安全需要注意两点：

1. 共享：意味着该资源可以由多个线程同时访问；
2. 可变：意味着该资源可以在其生命周期内被改修改。

由于每个线程执行的过程是不可控的，所以需要采用同步机制来协同对象可变状态的访问。 

由于java中的线程与操作系统的原生线程是一一对应的，所以当阻塞一个线程时，需要从用户态切换到内核态执行阻塞操作，这是很耗时的操作，synchronized 的使用就会导致上下文切换。

### synchronized同步方法

#### 实例方法

实例方法（类的非静态方法）同步是同步在拥有该方法的对象上的，即加锁会加在对象上。 

#### 静态方法

静态方法同步是同步在静态方法的类对象（不是类的实例，是指保存类信息的对象）上的，Java虚拟机中一个类只能对应一个类对象，所以只允许一个线程执行同一个类的静态同步方法。其实和实例方法同步是类似的，只不过实例方法同步是在类实例上加锁，静态方法同步是在类对象上加锁。  

#### 注意

1. 当一个线程正在访问一个对象的 synchronized 方法，那么其他线程不能访问该对象的其他 synchronized 方法。这个原因很简单，因为一个对象只有一把锁，当一个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，所以无法访问该对象的其他synchronized方法。
2. 当一个线程正在访问一个对象的 synchronized 方法，那么其他线程能访问该对象的非 synchronized 方法。这个原因很简单，访问非 synchronized 方法不需要获得该对象的锁，假如一个方法没用 synchronized 关键字修饰，说明它不会使用到临界资源，那么其他线程是可以访问这个方法的。
3. 如果一个线程 A 需要访问对象 object1 的 synchronized 方法 fun1，另外一个线程 B 需要访问对象 object2 的 synchronized 方法 fun1，即使 object1 和 object2 是同一类型），也不会产生线程安全问题，因为他们访问的是不同的对象，所以不存在互斥问题。

###synchronized同步块

#### 实例方法中的同步块

在同步块中，被括号括起来的对象叫做监视对象，即Java会给这个监视对象加锁以实现同步。 

#### 静态方法中的同步块

和实例方法中的同步块是一样的。  

synchronized 代码块类似于以下这种形式： 

~~~java
synchronized (lock){
    //访问共享可变资源
}
~~~

当在某个线程中执行这段代码块，该线程会获取对象lock的锁，从而使得其他线程无法同时访问该代码块。其中，lock 可以是 this，代表获取当前对象的锁，也可以是类中的一个属性，代表获取该属性的锁。特别地， 实例同步方法 与 synchronized(this)同步块 是互斥的，因为它们锁的是同一个对象。但与 synchronized(非this)同步块 是异步的，因为它们锁的是不同对象。

由此可见：synchronized代码块 比 synchronized方法 的粒度更细一些，使用起来也灵活得多 ，synchronized代码块可以实现只对需要同步的地方进行同步。 

### class对象锁

特别地，每个类也会有一个锁，静态的 synchronized方法 就是以Class对象作为锁。另外，它可以用来控制对 static 数据成员 （static 数据成员不专属于任何一个对象，是类成员） 的并发访问。并且，如果一个线程执行一个对象的非static synchronized 方法，另外一个线程需要执行这个对象所属类的 static synchronized 方法，也不会发生互斥现象。因为访问 static synchronized 方法占用的是类锁，而访问非 static synchronized 方法占用的是对象锁，所以不存在互斥现象。例如：

~~~java
public class Test {

    public static void main(String[] args)  {
        final InsertData insertData = new InsertData();
        new Thread(){
            @Override
            public void run() {
                insertData.insert();
            }
        }.start(); 
        new Thread(){
            @Override
            public void run() {
                insertData.insert1();
            }
        }.start();
    }  
}

class InsertData { 

    // 非 static synchronized 方法
    public synchronized void insert(){
        System.out.println("执行insert");
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("执行insert完毕");
    }

    // static synchronized 方法
    public synchronized static void insert1() {
        System.out.println("执行insert1");
        System.out.println("执行insert1完毕");
    }
}
/*
执行insert
执行insert1
执行insert1完毕
执行insert完毕
*/
~~~

###synchronized的内存语义

共享变量可见性问题主要是由于线程的工作内存导致的，进入synchronized块的内存语义是把synchronized块内使用到的变量从线程的工作内存中清除，这样synchronized块内使用到该变量就是直接从主内存中获取。推出synchronized块的内存语义是把synchronized块内对共享变量的修改刷新到主内存。

这也就是加锁和释放锁的语义

### synchronized的原理

Java的同步机制是通过对对象（资源）加锁实现的。例如： 
现在有一个对象A，线程T1和T2都会修改对象A的值。 首先假设线程T1比线程T2更早运行到修改对象A的代码。 
此时线程T1修改对象A的流程应该是这样的：线程T1从主存中获取对象A，并在自己缓存中存下对象A的副本，然后修改自己缓存中的对象A的副本。修改完毕后将自己缓存中的对象A的副本赋值给主存中的对象A，实现对象A的修改。 但是在线程T1执行修改对象A时，线程T2可能也跑到了修改对象A值的代码，而此时线程T1还在修改自己缓存中的对象A的副本，还没有将副本更新到主存中的对象A，但是线程T2又不知道，所以直接就从主存去取对象A了，这样就出现了竞态条件。 为了防止这种竞态条件的出现，就需要给对象A加锁。

下面看一下 synchronized的原理：

~~~java
public class InsertData {
    private Object object = new Object();

    public void insert(Thread thread){
        synchronized (object) {}
    }

    public synchronized void insert1(Thread thread){}

    public void insert2(Thread thread){}
}
~~~

上面这段代码反编译后的字节码为： 

![](synchronized\sync01.jpg)

从反编译获得的字节码可以看出，synchronized 代码块实际上多了 monitorenter 和 monitorexit 两条指令。 monitorenter指令执行时会让对象的锁计数加1，而monitorexit指令执行时会让对象的锁计数减1，其实这个与操作系统里面的PV操作很像，操作系统里面的PV操作就是用来控制多个进程对临界资源的访问。对于synchronized方法，执行中的线程识别该方法的 method_info 结构是否有 ACC_SYNCHRONIZED 标记设置，然后它自动获取对象的锁，调用方法，最后释放锁。如果有异常发生，线程自动释放锁。

有一点要注意：对于 synchronized方法 或者 synchronized代码块，当出现异常时，JVM会自动释放当前线程占用的锁，因此不会由于异常导致出现死锁现象。　　

synchronized 是可重入的，可重入锁最大的作用是避免死锁。

### 注意事项

1. 内置锁与字符串常量

   由于字符串常量池的原因，在大多数情况下，同步synchronized代码块 都不使用 String 作为锁对象，而改用其他，比如 new Object() 实例化一个 Object 对象，因为它并不会被放入缓存中。例：

   ~~~java
   //资源类
   class Service {
       public void print(String stringParam) {
           try {
               synchronized (stringParam) {
                   while (true) {
                       System.out.println(Thread.currentThread().getName());
                       Thread.sleep(1000);
                   }
               }
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
   }
   
   //线程A
   class ThreadA extends Thread {
       private Service service;
   
       public ThreadA(Service service) {
           super();
           this.service = service;
       }
   
       @Override
       public void run() {
           service.print("AA");
       }
   }
   
   //线程B
   class ThreadB extends Thread {
       private Service service;
   
       public ThreadB(Service service) {
           super();
           this.service = service;
       }
   
       @Override
       public void run() {
           service.print("AA");
       }
   }
   
   //测试
   public class Run {
       public static void main(String[] args) {
   
           //临界资源
           Service service = new Service();
   
           //创建并启动线程A
           ThreadA a = new ThreadA(service);
           a.setName("A");
           a.start();
   
           //创建并启动线程B
           ThreadB b = new ThreadB(service);
           b.setName("B");
           b.start();
   
       }
   }
   /* Output (死锁): 
           A
           A
           A
           A
           ...
   */
   ~~~

    出现上述结果就是因为 String 类型的参数都是 “AA”，两个线程持有相同的锁，所以 线程B 始终得不到执行，造成死锁。进一步地，所谓死锁是指：不同的线程都在等待根本不可能被释放的锁，从而导致所有的任务都无法继续完成。

2. 锁的是对象而非引用

   在将任何数据类型作为同步锁时，需要注意的是，是否有多个线程将同时去竞争该锁对象：  

   - 若它们将同时竞争同一把锁，则这些线程之间就是同步的；  
   - 否则，这些线程之间就是异步的。 

   例：

   ~~~java
   //资源类
   class MyService {
       private String lock = "123";
   
       public void testMethod() {
           try {
               synchronized (lock) {
                   System.out.println(Thread.currentThread().getName() + " begin "
                           + System.currentTimeMillis());
                   lock = "456";
                   Thread.sleep(2000);
                   System.out.println(Thread.currentThread().getName() + "   end "
                           + System.currentTimeMillis());
               }
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
   }
   
   //线程B
   class ThreadB extends Thread {
   
       private MyService service;
   
       public ThreadB(MyService service) {
           super();
           this.service = service;
       }
   
       @Override
       public void run() {
           service.testMethod();
       }
   }
   
   //线程A
   class ThreadA extends Thread {
   
       private MyService service;
   
       public ThreadA(MyService service) {
           super();
           this.service = service;
       }
   
       @Override
       public void run() {
           service.testMethod();
       }
   }
   
   //测试
   public class Run1 {
       public static void main(String[] args) throws InterruptedException {
   
           //临界资源
           MyService service = new MyService();
   
           //线程A
           ThreadA a = new ThreadA(service);
           a.setName("A");
   
           //线程B
           ThreadB b = new ThreadB(service);
           b.setName("B");
   
           a.start();
           Thread.sleep(50);// 存在50毫秒
           b.start();
       }
   }
   /* Output(循环): 
          A begin 1484319778766
          B begin 1484319778815
          A   end 1484319780766
          B   end 1484319780815
    */
   ~~~

   由上述结果可知，线程 A、B 是异步的。因为50毫秒过后， 线程B 取得的锁对象是 “456”，而 线程A 依然持有的锁对象是 “123”。所以，这两个线程是异步的。若将上述语句 “Thread.sleep(50);” 注释，则有： 

   ~~~java
   //测试
   public class Run1 {
       public static void main(String[] args) throws InterruptedException {
   
           //临界资源
           MyService service = new MyService();
   
           //线程A
           ThreadA a = new ThreadA(service);
           a.setName("A");
   
           //线程B
           ThreadB b = new ThreadB(service);
           b.setName("B");
   
           a.start();
           // Thread.sleep(50);// 存在50毫秒
           b.start();
       }
   }
   /* Output(循环): 
          B begin 1484319952017
          B   end 1484319954018
          A begin 1484319954018
          A   end 1484319956019
    */
   ~~~

   由上述结果可知，线程 A、B 是同步的。因为线程 A、B 竞争的是同一个锁“123”，虽然先获得运行的线程将 lock 指向了 对象“456”，但结果还是同步的。因为线程 A 和 B 共同争抢的锁对象是“123”，也就是说，**锁的是对象而非引用。** 

   ### 总结

   用一句话来说，synchronized 内置锁 是一种 对象锁 (锁的是对象而非引用)， 作用粒度是对象 ，可以用来实现对 临界资源的同步互斥访问 ，是可重入 的。特别地，对于临界资源 有：

   - 若该资源是静态的，即被 static 关键字修饰，那么访问它的方法必须是同步且是静态的，synchronized 块必须是 class锁；
   - 若该资源是非静态的，即没有被 static 关键字修饰，那么访问它的方法必须是同步的，synchronized 块是实例对象锁； 　

   实质上，关键字synchronized 主要包含两个特征：

   - 互斥性：保证在同一时刻，只有一个线程可以执行某一个方法或某一个代码块；
   - 可见性：保证线程工作内存中的变量与公共内存中的变量同步，使多线程读取共享变量时可以获得最新值的使用。