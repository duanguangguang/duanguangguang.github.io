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

#### 2. scheduleWithFixedDelay

#### 3. scheduleAtFixedRate

### 总结

ScheduledThreadPoolExecutor其内部使用DelayQueue来存放具体任务。任务分三种使用period的值来区分：

1. 一次性执行任务执行完毕就结束了；
2. fixed-delay任务保证同一个任务在多次执行之间间隔固定时间；
3. fixed-rate任务保证按照固定的频率执行。

![](ScheduleThreadPoolExecutor\stpe02.png)