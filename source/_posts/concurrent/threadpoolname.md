---
title: 优雅的使用线程池
date: 2019-01-27 17:28:48
tags:
 - 线程池
 - 面试
categories: 
 - 并发包
---

### 介绍

当在一个应用中需要创建多个线程或者线程池时最好给每个线程或者线程池根据业务类型设置具体的名称，以便在出现问题的时候方便进行定位。

<!-- more -->

### 线程名称

线程的名称如：Thread-0。我们看一下这个是怎么来的。

创建线程代码：

~~~java
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}
~~~

~~~java
private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize) {
    init(g, target, name, stackSize, null);
}
~~~

如上可知，如果调用没有指定线程名称的方法创建线程，其内部会使用“Thread-”+nextThreadNum()作为线程的默认名称：

~~~java
private static int threadInitNumber;
private static synchronized int nextThreadNum() {
    return threadInitNumber++;
}
~~~

由此可知，threadInitNumber是static变量，nextThreadNum是static方法，所以线程的编号是全应用唯一的并且是递增的。这里由于涉及多线程递增threadInitNumber，也就是执行读取-递增-写入操作，而这是线程不安全的，所以要使用方法级别的synchronized进行同步。

创建线程时给线程起名称的代码：

~~~java
static final String THREAD_SAVE_ORDER = "THREAD_SAVE_ORDER";
static final String THREAD_SAVE_ADDR = "THREAD_SAVE_ADDR";

public static void main(String[] args) {
    // 订单模块
    Thread threadOne = new Thread(()->{
        System.out.println("保存订单的线程");
        throw new NullPointerException();
    }, THREAD_SAVE_ORDER);

    // 发货模块
    Thread threadTwo = new Thread(()->{
        System.out.println("保存收获地址的线程");
    }, THREAD_SAVE_ADDR);

    threadOne.start();
    threadTwo.start();
}
~~~

### 线程池名称

线程的名称如：pool-1-thread-1。我们看一下这个是怎么来的。查看线程池创建的源码如下：

~~~java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
~~~

~~~java
public static ThreadFactory defaultThreadFactory() {
    return new DefaultThreadFactory();
}
~~~

~~~java
static class DefaultThreadFactory implements ThreadFactory {
    //(1)
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    //(2)
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    //(3)
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
        Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
            poolNumber.getAndIncrement() +
            "-thread-";
    }

    public Thread newThread(Runnable r) {
        //(4)
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
~~~

代码（1）中的poolNumber是static的原子变量，用来记录当前线程池的编号，它是应用级别的，所有线程池共用一个，比如创建第一个线程池时线程池编号是1，创建第二个线程池时线程池的编号是2，所以pool-1-thread-1中的pool-1中的1就是这个值。

代码（2）中的threadNumber是线程池级别的，每个线程池使用该变量来记录该线程池中线程的编号，所以pool-1-thread-1中的thread-1中的1就是这个值。

代码（3）中的namePrefix是线程池中线程名称的前缀，默认固定为pool。

代码（4）具体创建线程，线程的名称是使用namePrefix+threadNumber.getAndIncrement()拼接的。

由此，只需对DefaultThreadFactory的代码中的namePrefix的初始化做下手脚，即当需要创建线程池时传入与业务相关的namePrefix名称就可以了。代码如下：

~~~java
// 命名线程工厂
static class NameThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    NameThreadFactory(String name) {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
        Thread.currentThread().getThreadGroup();

        if (null ==name || name.isEmpty()) {
            name = "pool";
        }

        namePrefix = name +
            poolNumber.getAndIncrement() +
            "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
~~~

创建线程池的代码如下：

~~~java
static ThreadPoolExecutor executorOne = new ThreadPoolExecutor(5,
            5,
            1,
            TimeUnit.MINUTES,
            new LinkedBlockingDeque<>(),
            new NameThreadFactory("ASYN-ACCEPT-POOL"));
    
static ThreadPoolExecutor executorTwo = new ThreadPoolExecutor(5,
            5,
            1,
            TimeUnit.MINUTES,
            new LinkedBlockingDeque<>(),
            new NameThreadFactory("ASYN-PROCESS-POOL"));
~~~

### 总结

使用线程池的情况下当程序结束时需要调用shutdown关闭线程池。





