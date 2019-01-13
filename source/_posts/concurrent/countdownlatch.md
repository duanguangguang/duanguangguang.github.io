---
title: Java并发包中线程同步器原理解析
date: 2019-01-13 14:36:44
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

在日常开发中经常遇到需要在主线程中开启多个线程去并行执行任务，并且主线程需要等待所有子线程执行完毕后再进行汇总的场景，在CountDownLatch出现之前一直都使用join()方法来实现这一点，但是join 方法不够灵活，不能满足不同场景的需要，比如使用ExecutorService线程池来管理线程时传递的参数是Runable或者Callable对象，这时候就没办法直接调用线程的join方法，而是需要使用CountDownLatch了。

<!-- more -->

### CountDownLatch

#### CountDownLatch实现原理

CountDownLatch是使用AQS实现的，实际上是把计数器的值赋给了AQS的状态变量state。构造函数：

~~~java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
~~~

~~~java
Sync(int count) {
    setState(count);
}
~~~

1. void await()方法

   当线程调用CountDownLatch的await 方法后，当前线程会被阻塞，直到当所有线程都调用了CountDownLatch的countDown方法后，计数器的值为0时；或者其他线程调用了当前线程的interrupt()方法中断了当前线程就会抛出InterruptedException异常，然后返回。下面看await方法内部是如何调用AQS方法的：

   ~~~java
   public void await() throws InterruptedException {
       this.sync.acquireSharedInterruptibly(1);
   }
   ~~~

   await方法委托sync调用了AQS的acquireSharedInterruptibly方法：

   ~~~java
   // AQS获取共享资源时可被中断的方法
   public final void acquireSharedInterruptibly(int arg)
       throws InterruptedException {
       if (Thread.interrupted())
           throw new InterruptedException();
       // 查看当前计数器值是否为0，为0则直接返回，否则进入AQS的队列等待
       if (tryAcquireShared(arg) < 0)
           doAcquireSharedInterruptibly(arg);
   }
   ~~~

   sync类实现的AQS的接口：

   ~~~java
   protected int tryAcquireShared(int acquires) {
       return (getState() == 0) ? 1 : -1;
   }
   ~~~

   可以看到，这里的tryAcquireShared传递的arg参数没有被用到，调用tryAcquireShared的方法仅仅是为了检查当前状态值是不是为0，并没有调用CAS让当前状态值减1。

2. boolean await(long timeout, TimeUnit unit)方法

   带有超时时间的await方法：

   ~~~java
   public boolean await(long timeout, TimeUnit unit)
       throws InterruptedException {
       return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
   }
   ~~~

3. void countDown()方法

   线程调用该方法后，计数器值递减，递减后如果为0则唤醒所有因调用await方法而被阻塞的线程，否则什么都不做。下面看下countDown()方法是如何调用AQS的方法的。

   ~~~java
   public void countDown() {
       sync.releaseShared(1);
   }
   ~~~

   委托sync调用了AQS的releaseShared方法：

   ~~~java
   public final boolean releaseShared(int arg) {
       // 调用sync实现的tryReleaseShared
       if (tryReleaseShared(arg)) {
           // AQS释放资源的方法
           doReleaseShared();
           return true;
       }
       return false;
   }
   ~~~

   sync实现的tryReleaseShared：

   ~~~java
   protected boolean tryReleaseShared(int releases) {
       // 循环进行CAS，直到当前线程成功完成CAS使计数器值减1并更新到state
       for (;;) {
           int c = getState();
           // （1）如果当前状态值为0则直接返回
           if (c == 0)
               return false;
           // （2）使用CAS让计数器值减1 
           int nextc = c-1;
           if (compareAndSetState(c, nextc))
               return nextc == 0;
       }
   }
   ~~~

   代码（1）判断如果当前状态值为0则直接返回false，从而countDown()方法直接返回；否则执行代码（2）使用CAS将计数器减1，CAS失败则循环重试，否则如果当前计数器为0则返回true，返回true则说明是最后一个线程调用的countDown方法，那么该线程除了让计数器值减1外，还需要唤醒因调用await方法而被阻塞的线程，具体是调用AQS的doReleaseShared方法，来激活阻塞的线程。

   这里代码（1）貌似是多余的，其实不然，这是为了防止当计数器值为0后，其他线程有调用了countDown方法，如果没有代码（1）状态值就可能会变成负数。

4. long getCount()方法

   获取当前计数器的值，其内部还是调用了AQS的getState方法获取AQS的state的值，一般在测试时使用：

   ~~~java
   public long getCount() {
       return sync.getCount();
   }
   ~~~

   ~~~java
   int getCount() {
       // 调用了AQS的getState方法
       return getState();
   }
   ~~~

#### CountDownLatch使用

~~~java
public class JoinCountDownLatch {
    private static CountDownLatch  countDownLatch = new CountDownLatch();

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        // 将线程A添加到线程池
        executorService.submit(()->{
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
            System.out.println("child threadOne over!");
        });
        // 将线程B添加到线程池
        executorService.submit(()->{
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
            System.out.println("child threadTwo over!");
        });
        System.out.println("wait all child thread over!");
        // 等待子线程执行完毕，返回
        countDownLatch.await();
        System.out.println("all child thread over!");
        executorService.shutdown();
    }
}

// 当线程池个数小于countDownLatch计数个数时，返回结果正常
wait all child thread over!
child threadOne over!
child threadTwo over!
all child thread over!
// 当线程池个数大于或等于countDownLatch计数个数时，返回结果不正常
wait all child thread over!
child threadOne over!
all child thread over!
child threadTwo over!
~~~

主线程调用countDownLatch.await()方法后会被阻塞。子线程执行完毕后调用countDownLatch.countDown()方法让countDownLatch内部计数器减1，所以子线程执行完毕并且调用countDown方法后计数器会变成0，这时候主线程await啊方法才会返回。

countDownLatch与join的区别：

1. 调用一个子线程的join方法后，该线程会一直被阻塞直到子线程运行完毕，而countDownLatch则使用计数器来允许子线程运行完毕或者在运行中递减计数，也就是countDownLatch可以在子线程运行的任何时候让await方法返回而不一定必须等到线程结束；
2. countDownLatch相比join方法让线程同步有更灵活的控制。

#### 小结

CountDownLatch是使用AQS实现的，使用AQS的状态变量来存放计数器的值。首先在初始化CountDownLatch时设置状态值，当多个线程调用countDown方法时实际是原子性递减AQS的状态值。当线程调用await方法后当前线程会被放入AQS的阻塞队列等待计数器为0再返回，其他线程调用countdown方法让计数器递减1，当计数器值变为0时，当前线程还要调用AQS的doReleaseShared方法来激活由于调用await方法而被阻塞的线程。

### CyclicBarrier

####CyclicBarrier实现原理

#### CyclicBarrier使用

#### 小结



### Semaphore

####Semaphore实现原理

#### Semaphore使用

#### 小结

### 总结

### 参考文档



