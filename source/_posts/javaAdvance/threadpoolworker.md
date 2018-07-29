---
title: Java线程池的运行原理
date: 2018-03-17 20:45:11
categories:
 - JAVA
 - Java进阶
tags:
 - Java进阶
---

## 线程池的运行原理

本文基于JDK1.8源码进行分析。

线程池是维护了一批线程来处理用户提交的任务，达到线程复用的目的，线程池维护的这批线程被封装成了Worker。当我们向线程池提交任务时，通常使用execute方法，接下来就先从该方法开始分析。execute方法：

<!-- more -->

~~~java
public void execute(Runnable command) {
      if (command == null)
          throw new NullPointerException();
      //JDK8的源码中，线程池本身的状态跟worker数量使用同一个变量ctl来维护
      int c = ctl.get();
      //通过位运算得出当然线程池中的worker数量与构造参数corePoolSize进行比较
      if (workerCountOf(c) < corePoolSize) {
          //如果小于corePoolSize，则直接新增一个worker，并把当然用户提交的任务command作为参数，如果成功则返回。
          if (addWorker(command, true))
              return;
          //如果失败，则获取最新的线程池数据
          c = ctl.get();
      }
      //如果线程池仍在运行，则把任务放到阻塞队列中等待执行。
      if (isRunning(c) && workQueue.offer(command)) {
          //这里的recheck思路是为了处理并发问题
          int recheck = ctl.get();
          //当任务成功放入队列时，如果recheck发现线程池已经不再运行了则从队列中把任务删除
          if (! isRunning(recheck) && remove(command))
              //删除成功以后，会调用构造参数传入的拒绝策略。
              reject(command);
           //如果worker的数量为0（此时队列中可能有任务没有执行），则新建一个worker（由于此时新建woker的目的是执行队列中堆积的任务，
           //因此入参没有执行任务，详细逻辑后面会详细分析addWorker方法）。
          else if (workerCountOf(recheck) == 0)
              addWorker(null, false);
      }
      //如果前面的新增woker，放入队列都失败，则会继续新增worker，此时线程池的状态是woker数量达到corePoolSize，阻塞队列任务已满
      //只能基于maximumPoolSize参数新建woker
      else if (!addWorker(command, false))
          //如果基于maximumPoolSize新建woker失败，此时是线程池中线程数已达到上限，队列已满，则调用构造参数中传入的拒绝策略
          reject(command);
}
~~~

总结一下用户向线程池提交任务以后，线程池的执行逻辑：

- 如果当前woker数量小于corePoolSize，则新建一个woker并把当前任务分配给该woker线程，成功则返回。
- 如果第一步失败，则尝试把任务放入阻塞队列，如果成功则返回。
- 如果第二步失败，则判断如果当前woker数量小于maximumPoolSize，则新建一个woker并把当前任务分配给该woker线程，成功则返回。
- 如果第三步失败，则调用拒绝策略处理该任务。

从execute的源码可以看出addWorker方法是重中之重，马上来看下它的实现。addWorker方法：

~~~java
private boolean addWorker(Runnable firstTask, boolean core) {
        //这里有一段基于CAS+死循环实现的关于线程池状态，线程数量的校验与更新逻辑就先忽略了，重点看主流程。
        //...

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
             //把指定任务作为参数新建一个worker线程
            w = new Worker(firstTask);
            //这里是重点，咋一看，一定以为w.thread就是我们传入的firstTask
            //其实是通过线程池构造函数参数threadFactory生成的woker对象
            //也就是说这个变量t就是代表woker线程。绝对不是用户提交的线程任务firstTask！！！
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    //加锁之后仍旧是判断线程池状态等一些校验逻辑。
                    int rs = runStateOf(ctl.get());
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) 
                            throw new IllegalThreadStateException();
                        //把新建的woker线程放入集合保存，这里使用的是HashSet
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    //然后启动woker线程
                    //这里再强调一遍上面说的逻辑，该变量t代表woker线程，也就是会调用woker的run方法
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                //如果woker启动失败，则进行一些善后工作，比如说修改当前woker数量等等
                addWorkerFailed(w);
        }
        return workerStarted;
}
~~~

addWorker方法主要做的工作就是新建一个Woker线程，加入到woker集合中，然后启动该线程，那么接下来的重点就是Woker类的run方法了。worker执行方法：

~~~java
//Woker类实现了Runnable接口
public void run() {
            runWorker(this);
}

//最终woker执行逻辑走到了这里
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        //task就是Woker构造函数入参指定的任务，即用户提交的任务
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); 
        boolean completedAbruptly = true;
        try {
            //一般情况下，task都不会为空（特殊情况上面注释中也说明了），因此会直接进入循环体中
            //这里getTask方法是要重点说明的，它的实现跟我们构造参数设置存活时间有关
            //我们都知道构造参数设置的时间代表了线程池中的线程，即woker线程的存活时间，如果到期则回收woker线程，这个逻辑的实现就在getTask中。
            //来不及执行的任务，线程池会放入一个阻塞队列，getTask方法就是去阻塞队列中取任务，用户设置的存活时间，就是
            //从这个阻塞队列中取任务等待的最大时间，如果getTask返回null，意思就是woker等待了指定时间仍然没有
            //取到任务，此时就会跳过循环体，进入woker线程的销毁逻辑。
            while (task != null || (task = getTask()) != null) {
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    //该方法是个空的实现，如果有需要用户可以自己继承该类进行实现
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        //真正的任务执行逻辑
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        //该方法是个空的实现，如果有需要用户可以自己继承该类进行实现
                        afterExecute(task, thrown);
                    }
                } finally {
                    //这里设为null，也就是循环体再执行的时候会调用getTask方法
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            //当指定任务执行完成，阻塞队列中也取不到可执行任务时，会进入这里，做一些善后工作，比如在corePoolSize跟maximumPoolSize之间的woker会进行回收
            processWorkerExit(w, completedAbruptly);
        }
}
~~~

woker线程的执行流程就是首先执行初始化时分配给的任务，执行完成以后会尝试从阻塞队列中获取可执行的任务，如果指定时间内仍然没有任务可以执行，则进入销毁逻辑。**注：这里只会回收corePoolSize与maximumPoolSize直接的那部分woker**

## 初始化线程池时线程数的选择

- 如果任务是IO密集型，一般线程数需要设置2倍CPU数以上，以此来尽量利用CPU资源。
- 如果任务是CPU密集型，一般线程数量只需要设置CPU数加1即可，更多的线程数也只能增加上下文切换，不能增加CPU利用率。

上述只是一个基本思想，如果真的需要精确的控制，还是需要上线以后观察线程池中线程数量跟队列的情况来定。

## 线程池任务执行流程

1. 当线程池小于corePoolSize时，新提交任务将创建一个新线程执行任务，即使此时线程池中存在空闲线程。
2. 当线程池达到corePoolSize时，新提交任务将被放入workQueue中，等待线程池中任务调度执行
3. 当workQueue已满，且maximumPoolSize>corePoolSize时，新提交任务会创建新线程执行任务
4. 当提交任务数超过maximumPoolSize时，新提交任务由RejectedExecutionHandler处理
5. 当线程池中超过corePoolSize线程，空闲时间达到keepAliveTime时，关闭空闲线程
6. 当设置allowCoreThreadTimeOut(true)时，线程池中corePoolSize线程空闲时间达到keepAliveTime也将关闭

## 阻塞队列选择的区别

一般如果线程池任务队列采用LinkedBlockingQueue队列的话，那么不会拒绝任何任务（因为队列大小没有限制），这种情况下，ThreadPoolExecutor最多仅会按照最小线程数来创建线程，也就是说线程池大小被忽略了。

如果线程池任务队列采用ArrayBlockingQueue队列的话，那么ThreadPoolExecutor将会采取一个非常负责的算法，比如假定线程池的最小线程数为4，最大为8所用的ArrayBlockingQueue最大为10。随着任务到达并被放到队列中，线程池中最多运行4个线程（即最小线程数）。即使队列完全填满，也就是说有10个处于等待状态的任务，ThreadPoolExecutor也只会利用4个线程。如果队列已满，而又有新任务进来，此时才会启动一个新线程，这里不会因为队列已满而拒接该任务，相反会启动一个新线程。新线程会运行队列中的第一个任务，为新来的任务腾出空间。

这个算法背后的理念是：该池大部分时间仅使用核心线程（4个），即使有适量的任务在队列中等待运行。这时线程池就可以用作节流阀。如果挤压的请求变得非常多，这时该池就会尝试运行更多的线程来清理；这时第二个节流阀—最大线程数就起作用了。