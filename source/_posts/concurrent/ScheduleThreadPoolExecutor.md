---
title: Java并发包中线程池ScheduledThreadPoolExecutor原理解析
date: 2019-01-23 21:41:47
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

ThreadPoolExecutor是Executors工具类的一部分功能，另一部分是ScheduledThreadPoolExecutor的实现。ScheduledThreadPoolExecutor继承了ThreadPoolExecutor并实现了ScheduledExecutorService接口。线程池队列是DelayedWorkQueue，其和DelayedQueue类似，是一个延迟队列。

<!-- more -->

### 类图介绍

![](ScheduleThreadPoolExecutor\stpe01.png)

ScheduleFutureTask是具有返回值的任务，继承自FutureTask。FutureTask的内部有一个变量state用来表示任务的状态，一开始状态为NEW，所有状态为：

~~~java
private static final int NEW          = 0; // 初始状态
private static final int COMPLETING   = 1; // 执行中状态
private static final int NORMAL       = 2; // 正常运行结束状态
private static final int EXCEPTIONAL  = 3; // 运行中异常
private static final int CANCELLED    = 4; // 任务被取消
private static final int INTERRUPTING = 5; // 任务正在被取消
private static final int INTERRUPTED  = 6; // 任务已经被中断
~~~

可能的任务状态路径为：

- NEW->COMPLETING->NORMAL              初始状态->执行中->正常结束
- NEW->COMPLETING->EXCEPTIONAL      初始状态->执行中->执行异常
- NEW->CANCELLED                                     初始状态->任务取消
- NEW->INTERRUPTING->INTERRUPTED   初始状态->被中断中->被中断

ScheduleFutureTask内部还有一个变量period用来表示任务的类型，任务类型如下：

- period=0，说明当前任务是一次性的，执行完就退出了
- period为负数，说明当前任务为fixed-delay任务，是固定延迟的定时可重复执行任务
- period为正数，说明当前任务为fixed-rate任务，是固定频率的定时可重复执行任务

### 源码分析

ScheduledThreadPoolExecutor的构造函数：

~~~java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}

public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
~~~

由构造函数可知，线程池队列是DelayedWorkQueue。

#### 1. schedule

该方法的作用是提交一个延迟执行的任务，任务从提交时间算起延迟单位为unit的delay时间后开始执行。提交的任务不是周期性任务，任务只会执行一次。

~~~java
public ScheduledFuture<?> schedule(Runnable command,
                                   long delay,
                                   TimeUnit unit) {
    //（1）参数校验
    if (command == null || unit == null)
        throw new NullPointerException();
    //（2）任务转换
    RunnableScheduledFuture<?> t = decorateTask(command,
                new ScheduledFutureTask<Void>(command, null,                                                                     triggerTime(delay, unit)));
    //（3）添加任务到延迟队列
    delayedExecute(t);
    return t;
}
~~~

代码（2）装饰任务，把提交的command任务转换为ScheduledFutureTask。ScheduledFutureTask是具体放入延迟队列里面的东西。由于是延迟任务，所以ScheduledFutureTask实现了long getDelay(TimeUnit unit)和int compareTo(Delayed other)方法。triggerTime方法将延迟时间转换为绝对时间，也就是把当前时间的纳秒数加上延迟的纳秒数后的long型值。ScheduledFutureTask的构造函数：

~~~java
ScheduledFutureTask(Runnable r, V result, long ns) {
    //调用父类FutureTask的构造函数
    super(r, result);
    this.time = ns;
    //period为0，说明为一次性任务
    this.period = 0;
    this.sequenceNumber = sequencer.getAndIncrement();
}
~~~

父类FutureTask的构造函数：

~~~java
public FutureTask(Runnable runnable, V result) {
    // 通过适配器把runnable转换为callable
    this.callable = Executors.callable(runnable, result);
    // 设置当前任务状态为NEW
    this.state = NEW;       
}
~~~

long getDelay(TimeUnit unit)方法如下：用来计算当前任务还有多少时间就过期了。

~~~java
public long getDelay(TimeUnit unit) {
    // 装饰后时间-当前时间
    return unit.convert(time - now(), NANOSECONDS);
}
~~~

int compareTo(Delayed other)方法如下：

~~~java
public int compareTo(Delayed other) {
    if (other == this) // compare zero if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
    return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
}
~~~

compareTo的作用是加入元素到延迟队列后，在内部建立或者调整堆时会使用该元素的compareTo方法与队列里面其他元素进行比较，让最快要过期的元素放到对首。所以无论什么时候向队列里面添加元素，队首的元素都是最快要过期的元素。

将任务添加到延迟队列，delayedExecute方法如下：

~~~java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    //(4)如果线程池关闭了，则执行线程池拒绝策略
    if (isShutdown())
        reject(task);
    else {
        //（5）添加任务到延迟队列
        super.getQueue().add(task);
        //(6) 再次检查线程池状态
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            // 关闭则删除刚才添加的任务
            remove(task))
            // 该任务可能已经执行，需要取消
            task.cancel(false);
        else
            //（7）确保至少一个线程在处理任务
            ensurePrestart();
    }
}
~~~

如果代码（6）判断结果为false，则会执行代码（7）确保至少有一个线程在处理任务，即使核心线程数corePoolSize被设置为0。ensurePrestart代码如下：

~~~java
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    //增加核心线程数
    if (wc < corePoolSize)
        addWorker(null, true);
    //如果初始化corePoolSize==0，则也添加一个线程
    else if (wc == 0)
        addWorker(null, false);
}
~~~

下面看线程池里面的线程如何获取并执行执行任务。ScheduledFutureTask的run方法如下：

~~~java
public void run() {
    //(8)是否只执行一次
    boolean periodic = isPeriodic();
    //（9）取消任务
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    //（10）只执行一次，调用schedule方法时候
    else if (!periodic)
        ScheduledFutureTask.super.run();
    //（11）定时执行
    else if (ScheduledFutureTask.super.runAndReset()) {
        //（11.1）设置time = time + period
        setNextRunTime();
        //（11.2）重新加入该任务到delay队列
        reExecutePeriodic(outerTask);
    }
}
~~~

由于periodic的值为false，所以执行代码（10）调用父类FutureTask的run方法：

~~~java
public void run() {
    //(12)状态不是NEW直接返回
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    //(13)调用具体的callable的call方法执行任务
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                //（13.1）任务执行失败
                setException(ex);
            }
            //（13.2）任务执行成功则修改任务状态
            if (ran)
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
~~~

set方法如下：

~~~java
protected void set(V v) {
    //如果当前任务的状态为NEW，则设置为COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = v;
        //设置当前任务的状态为NORMAL，也就是任务正常结束
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
        finishCompletion();
    }
}
~~~

setException方法如下：

~~~java
protected void setException(Throwable t) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        //设置当前任务的状态为EXCEPTIONAL，也就是任务非正常结束
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
        finishCompletion();
    }
}
~~~

#### 2. scheduleWithFixedDelay

该方法的作用是当任务执行完毕后，让其延迟固定时间后再次运行（fixed-delay任务）。

~~~java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                 long initialDelay,
                                                 long delay,
    //（14）参数校验                                            TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    //（15）任务转换，注意这里是period=-delay<0
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(-delay));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    //(16)添加任务到队列                                            
    delayedExecute(t);
    return t;
}
~~~

这里会执行到代码（11），runAndReset方法如下：

~~~java
protected boolean runAndReset() {
    //（17）
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return false;
    //（18）
    boolean ran = false;
    int s = state;
    try {
        Callable<V> c = callable;
        if (c != null && s == NEW) {
            try {
                c.call(); // don't set result
                ran = true;
            } catch (Throwable ex) {
                setException(ex);
            }
        }
    } finally {
        runner = null;
        s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
    //（19）
    return ran && s == NEW;
}
~~~

任务正常执行完毕后不会设置任务的状态，这样做是为了让任务成为可重复执行的任务。代码（19）判断如果当前任务正常执行完毕并且任务状态为NEW则返回true，否则返回false。如果返回了true则执行代码（11.1）的setNextRunTime方法设置该任务下一次的执行时间：

~~~java
private void setNextRunTime() {
    long p = period;
    if (p > 0)
        //fixed-rate类型任务
        time += p;
    else
        //fixed-delay类型任务
        time = triggerTime(-p);
}
~~~

总结：fixed-delay类型的任务的执行原理为：当添加一个任务到延迟队列后，等待initalDelay时间，任务就会过期，过期的任务就会被从队列移除，并执行。执行完毕后，会重新设置任务的延迟时间，然后再把任务放入到延迟队列，循环往复。如果一个任务在执行中抛出了异常，那么这个任务就结束了，不会影响其他任务的执行。

#### 3. scheduleAtFixedRate

该方法相对起始时间以固定频率调用指定的任务（fixed-rate任务）。

~~~java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                              long initialDelay,
                                              long period,
                                              TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                      null,
                                      triggerTime(initialDelay, unit),
                                      unit.toNanos(period));
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
~~~

在讲fixed-rate类型的任务command转换为ScheduledFutureTask时设置period=period，不再是-period。所以当前任务执行完毕后，调用setNextRunTime设置任务下次执行的时间时执行的是time+=p而不再是time=triggerTime(-p)。

总结：fixed-rate方式执行规则为，时间为initdelday+n*period时启动任务，但是如果当前任务还没有完，下一次要执行的任务的时间到了，则不会并发执行，下次要执行的任务会延迟执行，要等到当前任务执行完毕后再执行。

### 总结

ScheduledThreadPoolExecutor其内部使用DelayQueue来存放具体任务。任务分三种使用period的值来区分：

1. 一次性执行任务执行完毕就结束了；
2. fixed-delay任务保证同一个任务在多次执行之间间隔固定时间；
3. fixed-rate任务保证按照固定的频率执行。

![](ScheduleThreadPoolExecutor\stpe02.png)