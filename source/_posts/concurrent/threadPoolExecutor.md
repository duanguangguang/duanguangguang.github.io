---
title: Java并发包中线程池ThreadPoolExecutor原理解析
date: 2019-01-16 22:17:47
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

线程池主要解决两个问题：

1. 当执行大量异步任务时线程池能够提供较好的性能。线程池中的线程是可以复用的，避免了线程创建和销毁带来的开销；
2. 线程池提供了一种资源限制和管理的手段，比如可以限制线程的个数，动态新增线程等ThreadPoolExecutor也保留了一些基本你的统计数据，比如当前线程池完成的任务数目等。

线程池提供了许多可调参数和可扩展性接口，以满足不同情境的需要。我们可以使用更方便的Executors的工厂方法，比如newCachedThreadPool（线程池线程个数最多可达`Integer.MAX_VALUE`，线程自动回收）、newFixedThreadPool（固定大小的线程池）和newSingleThreadExecutor（单个线程）等来创建线程池，用户还可以自定义线程池。

<!-- more -->

### 类图介绍

![](threadPoolExecutor\tpe01.png)

ThreadPoolExecutor继承了AbstractExecutorService，其中mainLock是独占锁，用来控制新增worker线程操作的原子性。termination是该锁对应的条件队列，在线程调用awaitTermination时用来存放阻塞的线程。

成员变量ctl是一个Integer的原子变量，用来记录线程池状态和线程池中线程个数，类似于ReentrantReadWriteLock使用一个变量来保存两种信息。

Worker继承AQS和Runnable接口，是具体承载任务的对象。自己实现了简单不可重入独占锁，其中state=0表示锁未被获取状态，state=1表示锁已经被获取的状态，state=-1是闯将worker时默认的状态，创建时状态设置为-1是为了避免该线程在运行runWorker()方法前被中断。其中firstTask记录该工作线程执行的第一个任务，thread是具体执行任务的线程。

DefaultThreadFactory是线程工厂，newThread方法是对线程的一个修饰。其中poolNumber是个静态的原子变量，用来统计线程工厂的个数，threadNumber用来记录每个线程工厂创建了多少线程。

假设Integer类型是32位二进制表示，则其中高3位用来表示线程池状态，后面29位用来记录线程池线程个数。

~~~java
// 默认是RUNNING状态，线程个数为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 线程个数掩码位数，具体平台下Integer的二进制位数-3后剩余位数所表示的数是线程的个数
private static final int COUNT_BITS = Integer.SIZE - 3;
// 线程最大个数（低29位）00011111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
~~~

### 线程池状态

~~~java
// 高3位 11100000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
// 高3位 00000000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 高3位 00100000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
// 高3位 01000000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
// 高3位 01100000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

// 获取高3位（运行状态）
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 获取低29位（线程个数）
private static int workerCountOf(int c)  { return c & CAPACITY; }
// 计算ctl新值（线程状态与线程个数）
private static int ctlOf(int rs, int wc) { return rs | wc; }
~~~

线程池状态含义：

- RUNNING：接受新任务并且处理阻塞队列里的任务
- SHUTDOWN：拒绝新任务但是处理阻塞队列里的任务
- STOP：拒绝心任务并且抛弃阻塞队列里的任务，同时会中断正在处理的任务
- TIDYING：所有任务都执行完（包含阻塞队列里面的任务）后当前线程池活动线程数为0，将要调用terminated方法
- TERMINATED：终止状态。terminated方法调用完之后的状态。

线程池状态转换：

- RUNNING->SHUTDOWN：显示调用shutdown()方法，或者隐式调用了finalize()方法里面的shutdown()方法
- RUNNING或SHUTDOWN->STOP：显示调用shutdownNow()方法时
- SHUTDOWN->TIDYING：当线程池和任务队列都为空时
- STOP->TIDYING：当线程池为空时
- TIDYING->TERMINATED：当terminated()hook方法执行完成时

### 线程池参数

ThreadPoolExecutor构造器：

~~~java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) { //...}
~~~

线程池参数如下：

- corePoolSize：线程池核心线程个数
- maximumPoolSize：线程池最大线程数量
- keepAliveTime：存活时间。如果当前线程池中的线程数量比核心线程数量多，并且是闲置状态，则这些闲置的线程能存活的最大时间
- unit：存活时间的时间单位
- workQueue：用于保存等待执行的任务的阻塞队列，比如基于数组的有界`ArrayBlockingQueue`、基于链表的无界`LinkedBlockingQueue`、最多只有一个元素的同步队列`SynchronousQueue`、优先队列`PriorityBlockingQueue`等
- threadFactory：创建线程的工厂
- handler：饱和策略，当队列满并且线程个数达到maximumPoolSize后采取的策略，比如AbortPolicy（抛出异常）、CallerRunsPolicy（使用调用者所在线程来运行任务）、DiscardOldestPolicy（调用poll丢弃一个任务，执行当前任务）、DiscardPolicy（默默丢弃，不抛出异常）

### 线程池类型

Executors是个工具类，里面提供了很多静态方法，根据用户选择返回不同的线程池实例。

#### 1. newFixedThreadPool

创建一个核心线程个数和最大线程个数都为nThreads的线程池，并且阻塞队列长度为Integer.MAX_VALUE。keepAliveTime=0说明只要线程个数比核心线程个数多并且当前空闲则回收。

~~~java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, 
                                  nThreads,
                                  0L, 
                                  TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
~~~

使用自定义线程创建工厂：

~~~java
public static ExecutorService newFixedThreadPool(int nThreads, 
                                                 ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, 
                                  nThreads,
                                  0L, 
                                  TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
~~~

#### 2. newSingleThreadExecutor

创建一个核心线程个数和最大线程个数都为1的线程池，并且阻塞队列长度为Integer.MAX_VALUE。

~~~java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 
                                1,
                                0L, 
                                TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
~~~

使用自定义线程创建工厂：

~~~java
public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 
                                1,
                                0L, 
                                TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>(),
                                threadFactory));
}
~~~

#### 3. newCachedThreadPool

创建一个按需创建线程的线程池，初始线程个数为0，最多线程个数为Integer.MAX_VALUE，并且阻塞队列为同步队列。keepAliveTime=60说明只要当前线程在60s内空闲则回收。这个类型的特殊之处在于，加入同步队列的任务会被马上执行，同步队列里面最多只有一个任务。

~~~java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, 
                                      Integer.MAX_VALUE,
                                      60L, 
                                      TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
~~~

使用自定义线程创建工厂：

~~~java
public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(0, 
                                  Integer.MAX_VALUE,
                                  60L, 
                                  TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>(),
                                  threadFactory);
}
~~~

### 源码分析

#### 1. 提交任务执行execute

execute方法的作用是提交任务command到线程池进行执行。用户线程提交任务到线程池的模型图如下所示：

![](threadPoolExecutor\tpe02.png)

如上图所示，ThreadPoolExecutor的实现实际是一个生产消费模型，当用户添加任务到线程池时相当于生产者生产元素，workers线程工作集中的线程直接执行任务或者从任务队列里面获取任务时则相当于消费者消费元素。

用户线程提交任务的execute方法如下：

~~~java
public void execute(Runnable command) {
    //(1)如果任务为null，则抛出NPE异常
    if (command == null)
        throw new NullPointerException();
    
    //（2）获取当前线程池的状态+线程个数变量的组合值
    int c = ctl.get();
    
    //（3）当前线程池中线程个数是否小于corePoolSize，小于则开启新线程运行
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    
    //（4）如果线程池处于RUNNING状态，则添加任务到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
        //（4.1）二次检查
        int recheck = ctl.get();
        //（4.2）如果当前线程池状态不是RUNNING则从队列中删除任务，并执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        //（4.3）否则如果当前线程池为空，则添加一个线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    
    //（5）如果队列满，则新增线程，新增失败则执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
~~~

 如果代码（4）添加任务失败，则说明任务队列已满，那么执行代码（5）尝试新开启线程来执行该任务，如果当前线程池中线程个数>maximumPoolSize则执行拒绝策略。

下面分析新增线程的addWorkder方法：

~~~java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // (6)检查队列是否只在必要时为空
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        //（7）循环CAS增加线程个数
        for (;;) {
            int wc = workerCountOf(c);
            //（7.1）如果线程个数超限则返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //（7.2）CAS增加线程个数，同时只有一个线程成功
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // （7.3）CAS失败了，则看线程池状态是否变化了，变化则跳到外层循环重新尝试获取线程池状态，否则内层循环重新CAS
            c = ctl.get();  
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    //（8）到这里说明CAS成功了
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        //（8.1）创建worker
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            //（8.2）加独占锁，为了实现workers同步，因为可能多个线程调用了线程池的execute方法
            mainLock.lock();
            try {
                //（8.3）重新检查线程池状态，以避免在获取锁前调用了shutdown接口
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //（8.4）添加任务
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //（8.5）添加成功后则启动任务
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
~~~

第一部分双重循环的目的是通过CAS操作增加线程数；第二部分主要是把并发安全的任务添加到workers里面，并且启动任务执行。代码（6）返回false的三种情况：

- 当前线程池状态为STOP、TIDYING或TERMINATED；
- 当前线程池状态为SHUTDOWM并且已经有了第一个任务；
- 当前线程池状态为SHUTDOWM并且任务队列为空。

内层循环的作用是使用CAS操作增加线程数，代码（7.1）判断如果线程个数超限则返回false，否则执行代码（7.2）CAS操作设置线程个数，CAS成功则退出双重循环，CAS失败则执行代码（7.3）看当前线程池的状态是否发生了变化，如果变化了，则再次进入外层循环重新获取线程池状态，都在进入内层循环继续进行CAS尝试。

代码（8）说明使用CAS成功地增加了线程个数，但是现在任务还没开始执行。这里使用全局的独占锁来控制把新增的Worker添加到工作集workers中。

#### 2. 工作线程Worker的执行

用户线程提交任务到线程池后，由Worker来执行。

~~~java
Worker(Runnable firstTask) {
    // 在调用runWorker前禁止中断
    setState(-1); 
    this.firstTask = firstTask;
    // 创建一个线程
    this.thread = getThreadFactory().newThread(this);
}
~~~

在构造函数内首先设置Worker的状态为-1，这是为了避免当前Worker在调用runWorker方法前被中断（当其他线程调用了线程池的shutdownNow时，如果Worker状态>=0则会中断该线程）。在下面runWorker方法中，运行代码（9）时会调用unlock方法，该方法吧status设置为了0.这时候调用shutdownNow会中断Worker线程。

~~~java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // (9)将state设置为0，允许中断
    w.unlock(); 
    boolean completedAbruptly = true;
    try {
        //（10）
        while (task != null || (task = getTask()) != null) {
            //（10.1）获取工作线程内部持有的独占锁
            w.lock();
            
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //（10.2）执行任务前干一些事情
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //（10.3）执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //（10.4）执行任务后干一些事情
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // （10.5）统计当前Worker完成了多少个任务
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //（11）执行清理工作
        processWorkerExit(w, completedAbruptly);
    }
}
~~~

这里在执行具体任务期间加锁，是为了避免在任务运行期间，其他线程调用了shutdown后正在执行的任务被中断，shutdown只会中断当前被阻塞挂起的线程。

~~~java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) 
        decrementWorkerCount();

    //(11.1)统计整个线程池完成的任务个数，并从工作集里面删除当前Worker
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    //(11.2)尝试设置线程池状态为TERMINATED，如果当前是SHUTDOWN状态并且工作队列为空，或者当前是STOP状态，当前线程池里面没有活动线程
    tryTerminate();

    //（11.3）如果当前线程个数小于核心个数，则增加一个线程
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
~~~

如果设置了TERMINATED状态，则还需要调用条件变量termination的signalAll方法激活所有因为调用线程池的awaitTermination方法而被阻塞的线程。

#### 3. shutdown操作

调用shutdown方法后，线程池就不会再接受新的任务了，但是工作队列里面的任务还是要执行的。该方法会立刻返回，并不等待队列任务完成再返回。

~~~java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //(12)权限检查
        checkShutdownAccess();
        //（13）设置当前线程池状态为SHUTDOWN，如果已经是SHUTDOWN则直接返回
        advanceRunState(SHUTDOWN);
        //（14）设置中断标志
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    //（15）尝试将状态变为TERMINATED
    tryTerminate();
}
~~~

代码（12）检查看是否设置了安全管理器，是则看当前调用shutdown命令的线程是否有关闭线程的权限，如果有权限则还要看调用线程是否有中断工作线程的权限，如果没有则抛出SecurityException或者NullPointerException异常。

代码（13）的内容是如果当前线程池状态>=SHUTDOWN则直接返回，否则设置为SHUTDOWN状态。

~~~java
private void advanceRunState(int targetState) {
    for (;;) {
        int c = ctl.get();
        if (runStateAtLeast(c, targetState) ||
            ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
            break;
    }
}
~~~

代码（14）其设置所有空闲线程的中断标志。这里首先加了全局锁，同时只有一个线程可以调用shutdown方法设置中断标志。然后尝试获取Worker自己的锁，获取成功则设置中断标志。由于正在执行的任务已经获取了锁，所以正在执行的任务没有被中断。这里中断的是阻塞到getTask方法并企图从队列里面获取任务的线程，也就是空闲线程。

~~~java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            // 如果工作线程没有被中断，并且没有正在运行则设置中断标志
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
~~~

~~~java
final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 设置当前线程池状态为TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        // 扩展接口terminated，在线程池状态变为TERMINATED前做一些事情
                        terminated();
                    } finally {
                        // 设置当前线程池状态为TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        // 激活因调用条件变量termination的await系列方法而被阻塞的所有线程
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
~~~

#### 4. shutdownNow操作

调用shutdownNow方法后，线程池就不会再接受新的任务了，并且会丢弃工作队列里面的任务，正在执行的任务会被中断，该方法会立刻返回，并不等待激活的任务执行完成。返回值为这时候队列里面被丢弃的任务列表。

~~~java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        //(16)权限检查
        checkShutdownAccess();
        //（17）设置线程池状态为STOP
        advanceRunState(STOP);
        //（18）中断所有线程
        interruptWorkers();
        //（19）将队列任务移动到tasks中
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
~~~

中断的所有线程包含空闲线程和正在执行任务的线程。

~~~java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
~~~

#### 5. awaitTermination操作

当线程调用awaitTermination方法后，当前线程会被阻塞，直到线程池状态变为TERMINATED才返回，或者等待时间超时后才返回。

~~~java
public boolean awaitTermination(long timeout, TimeUnit unit)
    throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (;;) {
            if (runStateAtLeast(ctl.get(), TERMINATED))
                return true;
            if (nanos <= 0)
                return false;
            nanos = termination.awaitNanos(nanos);
        }
    } finally {
        mainLock.unlock();
    }
}
~~~

首先获取独占锁，然后在无限循环内部判断当前线程池状态是否至少是TERMINATED状态，如果是则直接返回，都在说明当前线程池里面还有线程在执行，则看设置的超时时间nanos是否小于0，小于0则说明不需要等待，那就直接返回，如果大于0则调用变量termination的awaitNanos方法等待nanos时间，期望在这段时间内线程池状态变为TERMINATED。

在shutdown方法中，当线程池状态变为TERMINATED时会调用termination.signalAll()用来激活调用条件变量termination的await系列方法被阻塞的所有线程，所以如果在调用awaitTermination之后又调用了shutdown方法，并且在shutdown内部将线程池状态设置为TERMINATED，则termination.awaitNanos方法会返回。

在工作线程Worker的runWorker方法内，当工作线程运行结束后，会调用processWorkerExit方法，在processWorkerExit方法内部也会调用tryTerminate方法测试当前是否把线程池状态设置为TERMINATED，如果是，也会调用termination.signalAll()用来激活调用线程池的awaitTermination方法而被阻塞的线程。

当等待时间超时后，termination.awaitNanos也会返回，这时候重新检查当前线程池状态是否为TERMINATED，如果是则直接返回，否则继续阻塞挂起自己。

### 总结

线程池巧妙地使用一个Integer类型的原子变量来记录线程池状态和线程池中的线程个数。通过线程池状态来控制任务的执行，每个Worker线程可以处理多个任务。线程池通过线程的复用减少了线程创建和销毁的开销。