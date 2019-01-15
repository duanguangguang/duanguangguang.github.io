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

![](countdownlatch\cdl01.png)

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

CountDownLatch的计数器是一次性的，等到计数器值变为0后，再调用CountDownLatch的await和countdown方法都会立即返回，这起不到线程同步的效果。CyclicBarrier是回环屏障的意思，它可以让一组线程全部达到一个状态后再全部同时执行，当所有等待线程执行完毕，重置CyclicBarrier的状态，使其可以重复使用。之所以叫屏障是因为线程调用await方法后就会被阻塞，这个阻塞点就称为屏障点，等所有线程都调用了await方法后，线程们就会冲破屏障，继续向下运行。

####CyclicBarrier实现原理

![](countdownlatch\cdl02.png)

CyclicBarrier基于独占锁实现，本质底层还是基于AQS的。parties用来记录线程个数，这里表示多少线程调用await后，所有线程才会冲破屏障继续往下运行。而count一开始等于parties，每当有线程调用await方法就递减1，当count为0时就表示所有线程都到了屏障点。使用两个变量（parties和count）的原因是，parties始终用来记录总的线程个数，当count计数器值变为0后，会将parties的值赋给count，从而复用。这两个变量是在构造CyclicBarrier对象时传递的。

~~~java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
~~~

barrierCommand是一个任务，这个任务的执行时机是当所有线程都到达屏障点后，使用lock首先保证了更新计数器count的原子性。另外使用lock的条件变量trip支持线程间使用await和signal操作进行同步。最后，在变量generation内部有一个变量broken，其用来记录当前屏障是否被打破，这里的broken没有被声明为volatile因为是在锁内使用变量，所以不需要声明。

1. int await()方法

   当线程调用CyclicBarrier的await方法时会被阻塞，直到满足下面条件之一才会返回：

   - parties个线程都调用了await方法，线程都到了屏障点；
   - 其他线程调用了当前线程的interrupt方法中断了当前线程；
   - 与当前屏障点关联的Generation对象的broken标志被设置为true时，会抛出BrokenBarrierException异常，然后返回。

   其内部调用了dowait方法：第一个参数为false表示不设置超时时间

   ~~~java
   public int await() throws InterruptedException, BrokenBarrierException {
       try {
           return dowait(false, 0L);
       } catch (TimeoutException toe) {
           throw new Error(toe); // cannot happen
       }
   }
   ~~~

2. boolean await(long timeout, TimeUnit unit)方法

   设置了超时时间的await方法：

   ~~~java
   public int await(long timeout, TimeUnit unit)
       throws InterruptedException,
       BrokenBarrierException,
       TimeoutException {
       return dowait(true, unit.toNanos(timeout));
   }
   ~~~

3. int dowait(boolean timed, long nanos)方法

   该方法实现了CyclicBarrier的核心功能，其代码如下：

   ~~~java
   private int dowait(boolean timed, long nanos)
       throws InterruptedException, BrokenBarrierException,
   TimeoutException {
       final ReentrantLock lock = this.lock;
       lock.lock();
       try {
           final Generation g = generation;
   
           if (g.broken)
               throw new BrokenBarrierException();
   
           if (Thread.interrupted()) {
               breakBarrier();
               throw new InterruptedException();
           }
   
           // (1)如果index==0则说明所有线程都到了屏障点，此时执行初始化时传递的任务
           int index = --count;
           if (index == 0) {  // tripped
               boolean ranAction = false;
               try {
                   final Runnable command = barrierCommand;
                   // （2） 执行任务
                   if (command != null)
                       command.run();
                   ranAction = true;
                   // （3）激活其他因调用await方法而被阻塞的线程，并重置CyclicBarrier
                   nextGeneration();
                   return 0;
               } finally {
                   if (!ranAction)
                       breakBarrier();
               }
           }
   
           // （4）如果index!=0
           for (;;) {
               try {
                   // （5）没有设置超时时间
                   if (!timed)
                       trip.await();
                   // （6）设置了超时时间
                   else if (nanos > 0L)
                       nanos = trip.awaitNanos(nanos);
               } catch (InterruptedException ie) {
                   if (g == generation && ! g.broken) {
                       breakBarrier();
                       throw ie;
                   } else {
                       // We're about to finish waiting even if we had not
                       // been interrupted, so this interrupt is deemed to
                       // "belong" to subsequent execution.
                       Thread.currentThread().interrupt();
                   }
               }
   
               if (g.broken)
                   throw new BrokenBarrierException();
   
               if (g != generation)
                   return index;
   
               if (timed && nanos <= 0L) {
                   breakBarrier();
                   throw new TimeoutException();
               }
           }
       } finally {
           lock.unlock();
       }
   }
   ~~~

   ~~~java
   private void nextGeneration() {
       // （7） 唤醒条件队列里面阻塞线程
       trip.signalAll();
       // （8） 重置CyclicBarrier
       count = parties;
       generation = new Generation();
   }
   
   ~~~

   当一个线程调用了dowait方法后，首先会获取独占锁lock，如果创建CyclicBarrier时传递的参数为10，那么后面9个调用线程会被阻塞。然后当前获取到锁的线程会对计数器count进行递减操作，递减后count=index=9，因为index!=0所以当前线程会执行代码（4）.如果当前线程调用的是无参数的await方法，则这里timed=false，所以当前线程会被放入条件变量trip的条件阻塞队列，当前线程会被挂起并释放获取的lock锁。如果调用的是有参数的await方法则timed=true，然后当前线程也会被放入条件变量的条件队列并释放锁资源，不同的是当前线程会在指定时间超时后自动被激活。其他9个线程同理。最后count=index=0，所以执行代码（2），如果创建CyclicBarrier时传递了任务，则在其他线程被唤醒前先执行任务，任务执行完毕后再执行代码（3），唤醒其他9个线程，并重置CyclicBarrier，然后这10个线程就可以继续向下运行了。

#### CyclicBarrier使用

例：假设一个任务由阶段1、阶段2、阶段3组成，每个线程要串行地执行阶段123。

~~~java
public class CyclicBarrierTest {
    private static CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

    public static void main (String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);

        // 将线程A添加到线程池
        executorService.submit(()->{
            try {
                System.out.println(Thread.currentThread() + "step1");
                cyclicBarrier.await();

                System.out.println(Thread.currentThread() + "step2");
                cyclicBarrier.await();

                System.out.println(Thread.currentThread() + "step3");
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        // 将线程B添加到线程池
        executorService.submit(()->{
            try {
                System.out.println(Thread.currentThread() + "step1");
                cyclicBarrier.await();

                System.out.println(Thread.currentThread() + "step2");
                cyclicBarrier.await();

                System.out.println(Thread.currentThread() + "step3");
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        executorService.shutdown();
    }
}

// 输出结果
Thread[pool-1-thread-1,5,main]step1
Thread[pool-1-thread-2,5,main]step1
Thread[pool-1-thread-2,5,main]step2
Thread[pool-1-thread-1,5,main]step2
Thread[pool-1-thread-1,5,main]step3
Thread[pool-1-thread-2,5,main]step3
~~~

每个子线程在执行完阶段1后都调用了await方法，等到所有线程都达到屏障点后才会一块往下执行，这样保证了阶段1执行完了后才会执行阶段2...

#### 小结

CyclicBarrier通过独占锁ReentrantLock实现计数器原子性更新，并使用条件变量队列来实现线程同步。

### Semaphore

Semaphore信号量也是Java中的一个同步器，与CountDownLatch和CyclicBarrier不同的是，它内部的计数器是递增的，并且在一开始初始化Semaphore时可以指定一个初始值，但是并不需要知道需要同步的线程个数，而是在需要同步的地方调用acquire方法时指定需要同步的线程个数。

####Semaphore实现原理

![](countdownlatch\cdl03.png)

由类图可知，Semaphore还是使用AQS实现的。Sync有两个实现类，用来指定获取信号量时是否采用公平策略。

~~~java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
~~~

~~~java
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
~~~

~~~java
Sync(int permits) {
    setState(permits);
}
~~~

Semaphore默认采用非公平策略，这里AQS的state值也表示当前持有的信号量个数。

1. void acquire()方法

   当线程调用该方法的目的是希望获取一个信号量资源。如果当前信号量个数大于0，则当前信号量的计数会减1，然后该方法直接返回。否则如果当前信号量个数等于0，则当前线程会被放入AQS的阻塞队列。当其他线程调用了当前线程的interrupt()方法中断了当前线程时，则当前线程会抛出异常返回。

   ~~~java
   public void acquire() throws InterruptedException {
       // 传递参数为1，说明要获取1个信号量资源
       sync.acquireSharedInterruptibly(1);
   }
   ~~~

   ~~~java
   public final void acquireSharedInterruptibly(int arg)
       throws InterruptedException {
       if (Thread.interrupted())
           throw new InterruptedException();
       // 调用Sync子类方法尝试获取，这里根据构造函数确定使用公平还是非公平策略
       if (tryAcquireShared(arg) < 0)
           // 如果获取失败则放入阻塞队列，然后再次尝试，如果失败则调用park方法挂起当前线程
           doAcquireSharedInterruptibly(arg);
   }
   ~~~

   尝试获取信号量资源的AQS的方法tryAcquireShared是由Sync的子类实现的，所以这里涉及到两种策略：

   - 公平策略

     NonfairSync类的tryAcquireShared：

     ~~~java
     protected int tryAcquireShared(int acquires) {
         return nonfairTryAcquireShared(acquires);
     }
     ~~~

     ~~~java
     final int nonfairTryAcquireShared(int acquires) {
         for (;;) {
             // 获取当前信号量值
             int available = getState();
             // 计算当前剩余值
             int remaining = available - acquires;
             // 如果当前剩余值小于0或者CAS设置成功则返回
             if (remaining < 0 ||
                 compareAndSetState(available, remaining))
                 return remaining;
         }
     }
     ~~~

     如果剩余值小于0则说明当前信号量个数满足不了需求，那么直接返回负数，这时当前线程会被放入AQS的阻塞队列而被挂起。如果剩余值大于0，则使用CAS操作设置当前信号量值为剩余值然后返回剩余值。

   - 非公平策略

     ~~~java
     protected int tryAcquireShared(int acquires) {
         for (;;) {
             if (hasQueuedPredecessors())
                 return -1;
             int available = getState();
             int remaining = available - acquires;
             if (remaining < 0 ||
                 compareAndSetState(available, remaining))
                 return remaining;
         }
     }
     ~~~

     公平性还是靠hasQueuedPredecessors这个函数来保证的。公平策略是看当前线程节点的前驱节点是否也在等待获取该资源，如果是则自己放弃获取的权限，然后当前线程会被放入AQS阻塞队列，否则就去获取。

2. void acquire(int permits)方法

   该方法是获取permits个信号量。

   ~~~java
   public void acquire(int permits) throws InterruptedException {
       if (permits < 0) throw new IllegalArgumentException();
       sync.acquireSharedInterruptibly(permits);
   }
   ~~~

3. void acquireUninterruptibly()方法

   与acquire()方法类似，不响应中断。

   ~~~java
   public void acquireUninterruptibly() {
       sync.acquireShared(1);
   }
   ~~~

4. void acquireUninterruptibly(int permits)方法

   与acquire(int permits)方法类似，不响应中断。

   ~~~java
   public void acquireUninterruptibly(int permits) {
       if (permits < 0) throw new IllegalArgumentException();
       sync.acquireShared(permits);
   }
   ~~~

5. void release()方法

6. 该方法的作用是把当前Semaphore对象的信号量值增加1，如果当前有线程因为调用acquire方法被阻塞而被放入了AQS的阻塞队列，则会根据公平（或非公平）策略选择一个信号量个数能被满足的线程进行激活，激活的线程会尝试获取刚增加的信号量。

   ~~~java
   public void release() {
       sync.releaseShared(1);
   }
   ~~~

   ~~~java
   public final boolean releaseShared(int arg) {
       // （2）尝试释放资源
       if (tryReleaseShared(arg)) {
           // （3）资源释放成功则调用park方法唤醒AQS队列里面最先挂起的线程
           doReleaseShared();
           return true;
       }
       return false;
   }
   ~~~

   ~~~java
   protected final boolean tryReleaseShared(int releases) {
       for (;;) {
           int current = getState();
           int next = current + releases;
           if (next < current) // overflow
               throw new Error("Maximum permit count exceeded");
           // 使用CAS保证更新信号量值的原子性
           if (compareAndSetState(current, next))
               return true;
       }
   }
   ~~~

   tryReleaseShared方法增加信号量值成功后会执行代码（3），即调用AQS的方法来激活因为调用acquire方法而被阻塞的线程。

7. void release(int permits)方法

   每次调用会在信号量原来的基础上增加permits。

   ~~~java
   public void release(int permits) {
       if (permits < 0) throw new IllegalArgumentException();
       sync.releaseShared(permits);
   }
   ~~~

   可以看到，这里的sync.releaseShared是共享方法，这说明该信号量是线程共享的，信号量没有和固定线程绑定，多个线程可以同时使用CAS去更新信号量的值而不会被阻塞。

#### Semaphore使用

模拟CyclicBarrier复用功能。

~~~java
public class SemaphoreTest {
    private static volatile Semaphore semaphore = new Semaphore(0);

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(2);

        // 将线程A添加到线程池
        executorService.submit(()->{
            try {
                System.out.println(Thread.currentThread() + " A task over");
                semaphore.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        // 将线程B添加到线程池
        executorService.submit(()->{
            try {
                System.out.println(Thread.currentThread() + " A task over");
                semaphore.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        // (1)等待子线程执行任务A完毕
        semaphore.acquire(2);
        System.out.println("task A is over");

        // 将线程C添加到线程池
        executorService.submit(()->{
            try {
                System.out.println(Thread.currentThread() + " B task over");
                semaphore.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        // 将线程D添加到线程池
        executorService.submit(()->{
            try {
                System.out.println(Thread.currentThread() + " B task over");
                semaphore.release();
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        // (2)等待子线程执行任务B完毕
        semaphore.acquire(2);
        System.out.println("task B is over");
        executorService.shutdown();
    }
}

// 输出结果
Thread[pool-1-thread-1,5,main] A task over
Thread[pool-1-thread-2,5,main] A task over
task A is over
Thread[pool-1-thread-1,5,main] B task over
Thread[pool-1-thread-2,5,main] B task over
task B is over
~~~

可以看出Semaphore在某种程度上实现了CyclicBarrier的复用功能。

#### 小结

Semaphore完全可以达到CountDownLatch的效果，但是Semaphore的计数器也是不可以自动重置的，不过可以通过变相地改变acquire方法的参数还是可以实现CyclicBarrier的功能的。

### 总结

使用同步器会大大减少wait、notify等来实现线程同步，在日常开发中当需要进行线程同步时使用这些同步类会节省很多代码并且可以保证正确性。

### 参考文档

1. 《Java并发编程之美》

