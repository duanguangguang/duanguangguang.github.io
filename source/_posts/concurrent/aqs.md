---
title: 抽象同步队列AQS
date: 2018-12-23 13:41:06
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

**AQS**是抽象同步队列AbstractQueuedSynchronizer的简称，AbstractQueuedSynchronizer 是JUC 中通过 Sync Queue(并发安全的 CLH Queue), Condition Queue(普通的 list) , volatile 变量 state 提供的 控制线程获取统一资源(state) 的 Synchronized 工具，是实现同步器的基础组件，也是并发包中锁的底层的实现。

AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，如ReentrantLock、Semaphore、CountDownLatch等等。

工作机制：AQS的等待队列是基于链表实现的FIFO的等待队列，队列每个节点只关心其前驱节点的状态，线程唤醒时只唤醒队头等待线程（即head的后继节点，并且等待状态不大于0）。

<!-- more -->

### 主要特点

1. 内部含有两条 Queue(Sync Queue, Condition Queue) ；
2. AQS 内部定义获取锁(acquire)，释放锁(release)的主逻辑，子类实现响应的模版方法即可 
3. 支持共享和独占两种模式（共享模式时只用 Sync Queue，独占模式有时只用 Sync Queue，但若涉及 Condition，则还有 Condition Queue）；独占是排他的。
4. 支持 不响应中断获取独占锁(acquire)，响应中断获取独占锁(acquireInterruptibly)，超时获取独占锁(tryAcquireNanos)；不响应中断获取共享锁(acquireShared)，响应中断获取共享锁(acquireSharedInterruptibly)，超时获取共享锁(tryAcquireSharedNanos)；
5. 在子类的 tryAcquire，tryAcquireShared 中实现公平与非公平的区分。

### 类图结构

![](aqs\aqs01.png)

由此可以看到，AQS是一个FIFO的双向队列，其内部通过节点head和tail记录队首和队尾元素，队列元素的类型为Node。类图说明如下。

Node节点：

1. 其中Node中的thread变量用来存放进入AQS队列里面的线程；
2. Node节点内部的SHARED用来标记该线程是获取共享资源时被阻塞挂起后放入AQS队列的；
3. Node节点内部的EXCLUSIVE用来标记该线程是获取独占资源时被挂起后放入AQS队列的；
4. Node中的waitStatus记录当前线程等待状态，可以为：
   - CANCELIED：线程被取消了
   - SIGNAL：线程需要被唤醒
   - CONDITION：线程在条件队列里面等待
   - PROPAGATE：释放共享资源时需要通知其他节点
5. Node中的prev记录当前节点的前驱节点，next记录当前节点的后继节点。

AQS：

1. AQS中维护了一个volatile int state，代表共享资源，state访问方式有三种：
   - getState()
   - setState()
   - compareAndSetState()
2. 对于ReentrantLock来说，state用来表示当前线程获取锁的可重入次数；
3. 对于ReentrantReadWriteLock来说，state的高16位表示读状态，即获取该读锁的次数，低16位表示获取到写锁的线程的可重入次数；
4. 对于semaphore来说，state表示当前可用信号的个数；
5. 对于CountDownlatch来说，state表示计数器当前的值；
6. AQS中维护了一个FIFO线程等待队列，多线程争用资源被阻塞时会进入此队列。

条件变量ConditionObject:

1. ConditionObject用来结合锁实现线程同步，ConditionObject可以直接访问AQS对象内部的state状态值以及AQS队列；
2. 每个条件变量对应一个条件队列（单向列表），用来存放调用条件变量的await方法后被阻塞的线程；
3.  用于独占的模式，主要是线程释放lock，加入 Condition Queue，并进行相应的 signal 操作。

Queue：

1. Condition Queue，这个队列是用于独占模式中，只有用到 Condition.awaitXX 时才会将 node加到 tail 上(PS: 在使用 Condition的前提是已经获取 Lock)
2. Sync Queue，独占共享的模式中均会使用到的存放 Node 的 CLH queue，主要特点是，队列中总有一个 dummy 节点，后继节点获取锁的条件由前继节点决定, 前继节点在释放 lock 时会唤醒sleep中的后继节点

对AQS来说，线程同步的关键是对状态值state进行操作，根据state是否属于一个线程，AQS定义两种同步方式：

- Exclusive：独占，只有一个线程能执行，如ReentrantLock；
- Share：共享，多个线程可同时执行，如Semaphore/CountDownLatch。

共享模式时只用 Sync Queue，独占模式有时只用 Sync Queue，但若涉及 Condition，则还有 Condition Queue。在子类的 tryAcquire，tryAcquireShared 中实现公平与非公平的区分。

### 内部类Node

~~~java
static final class Node {
    /** 标识节点是否是 共享的节点(这样的节点只存在于 Sync Queue 里面) */
    static final Node SHARED = new Node();
    //独占模式
    static final Node EXCLUSIVE = null;
    /**
     *  CANCELLED 说明节点已经 取消获取 lock 了(一般是由于 interrupt 或 timeout 导致的)
     *  很多时候是在 cancelAcquire 里面进行设置这个标识
     */
    static final int CANCELLED = 1;

    /**
     * SIGNAL 标识当前节点的后继节点需要唤醒(PS: 这个通常是在 独占模式下使用, 在共享模式下有时用 PROPAGATE)
     */
    static final int SIGNAL = -1;
    
    //当前节点在 Condition Queue 里面
    static final int CONDITION = -2;
    
    /**
     * 当前节点获取到 lock 或进行 release lock 时, 共享模式的最终状态是 PROPAGATE(PS: 有可能共享模式的节点变成 PROPAGATE 之前就被其后继节点抢占 head 节点, 而从Sync Queue中被踢出掉)
     */
    static final int PROPAGATE = -3;

    volatile int waitStatus;

    /**
     * 节点在 Sync Queue 里面时的前继节点(主要来进行 skip CANCELLED 的节点)
     * 注意: 根据 addWaiter方法:
     *  1. prev节点在队列里面, 则 prev != null 肯定成立
     *  2. prev != null 成立, 不一定 node 就在 Sync Queue 里面
     */
    volatile Node prev;

    /**
     * Node 在 Sync Queue 里面的后继节点, 主要是在release lock 时进行后继节点的唤醒
     * 而后继节点在前继节点上打上 SIGNAL 标识, 来提醒他 release lock 时需要唤醒
     */
    volatile Node next;

    //获取 lock 的引用
    volatile Thread thread;

    /**
     * 作用分成两种:
     *  1. 在 Sync Queue 里面, nextWaiter用来判断节点是 共享模式, 还是独占模式
     *  2. 在 Condition queue 里面, 节点主要是链接且后继节点 (Condition queue是一个单向的, 不支持并发的 list)
     */
    Node nextWaiter;

    // 当前节点是否是共享模式
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 获取 node 的前继节点
    final Node predecessor() throws NullPointerException{
        Node p = prev;
        if(p == null){
            throw new NullPointerException();
        }else{
            return p;
        }
    }

    Node(){
        // Used to establish initial head or SHARED marker
    }

    // 初始化 Node 用于 Sync Queue 里面
    Node(Thread thread, Node mode){     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    //初始化 Node 用于 Condition Queue 里面
    Node(Thread thread, int waitStatus){ // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
~~~

waitStatus的状态变化:

1. 线程刚入 Sync Queue 里面, 发现独占锁被其他人获取, 则将其前继节点标记为 SIGNAL, 然后再尝试获取一下锁(调用 tryAcquire 方法)
2. 若调用 tryAcquire 方法获取失败, 则判断一下是否前继节点被标记为 SIGNAL, 若是的话 直接 block(block前会确保前继节点被标记为SIGNAL, 因为前继节点在进行释放锁时根据是否标记为 SIGNAL 来决定唤醒后继节点与否 <- 这是独占的情况下)
3. 前继节点使用完lock, 进行释放, 因为自己被标记为 SIGNAL, 所以唤醒其后继节点

waitStatus 变化过程:

1. 独占模式下:  0(初始) -> signal(被后继节点标记为release需要唤醒后继节点) -> 0 (等释放好lock, 会恢复到0)
2. 独占模式 + 使用 Condition情况下: 0(初始) -> signal(被后继节点标记为release需要唤醒后继节点) -> 0 (等释放好lock, 会恢复到0)其上可能涉及 中断与超时, 只是多了一个 CANCELLED, 当节点变成 CANCELLED, 后就等着被清除。
3. 共享模式下: 0(初始) -> PROPAGATE(获取 lock 或release lock 时) (获取 lock 时会调用 setHeadAndPropagate 来进行 传递式的唤醒后继节点, 直到碰到 独占模式的节点)
4. 共享模式 + 独占模式下: 0(初始) -> signal(被后继节点标记为release需要唤醒后继节点) -> 0 (等释放好lock, 会恢复到0)

其上的这些状态变化主要在: doReleaseShared , shouldParkAfterFailedAcquire 里面。

### Sync Queue

AQS内部维护着一个FIFO的CLH队列，所以AQS并不支持基于优先级的同步策略。至于为何要选择CLH队列，主要在于CLH锁相对于MSC锁，他更加容易处理cancel和timeout，同时他具备进出队列快、无所、畅通无阻、检查是否有线程在等待也非常容易（head != tail,头尾指针不同）。当然相对于原始的CLH队列锁，ASQ采用的是一种变种的CLH队列锁：

1. 原始CLH使用的locked自旋，而AQS的CLH则是在每个node里面使用一个状态字段来控制阻塞，而不是自旋。
2. 为了可以处理timeout和cancel操作，每个node维护一个指向前驱的指针。如果一个node的前驱被cancel，这个node可以前向移动使用前驱的状态字段。
3. head结点使用的是傀儡结点。

![](aqs\aqs03.png)

这个图代表有个线程获取lock, 而 Node1, Node2, Node3 则在Sync Queue 里面进行等待获取lock（PS: 注意到 dummy Node 的SINGNAL 这是叫获取 lock 的线程在释放lock时通知后继节点的标示）。

1. Sync Queue节点入队方法：

   下面独占方式和共享方式也是使用的Sync Queue，出队入队用的都是下面这个方法。

   ~~~java
   private Node addWaiter(Node mode){
       Node node = new Node(Thread.currentThread(), mode);      // 1. 封装 Node
       Node pred = tail;
       if(pred != null){                           // 2. pred != null -> 队列中已经有节点, 直接 CAS 到尾节点
           node.prev = pred;                       // 3. 先设置 Node.pre = pred (PS: 则当一个 node在Sync Queue里面时  node.prev 一定 != null(除 dummy node), 但是 node.prev != null 不能说明其在 Sync Queue 里面, 因为现在的CAS可能失败 )
           if(compareAndSetTail(pred, node)){      // 4. CAS node 到 tail
               pred.next = node;                  // 5. CAS 成功, 将 pred.next = node (PS: 说明 node.next != null -> 则 node 一定在 Sync Queue, 但若 node 在Sync Queue 里面不一定 node.next != null)
               return node;
           }
       }
       enq(node);                                 // 6. 队列为空, 调用 enq 入队列
       return node;
   }
   
   
   /**
    * 这个插入会检测head tail 的初始化, 必要的话会初始化一个 dummy 节点, 这个和 ConcurrentLinkedQueue 一样的
    * 将节点 node 加入队列
    * 这里有个注意点
    * 情况:
    *      1. 首先 queue是空的
    *      2. 初始化一个 dummy 节点
    *      3. 这时再在tail后面添加节点(这一步可能失败, 可能发生竞争被其他的线程抢占)
    *  这里为什么要加入一个 dummy 节点呢?
    *      这里的 Sync Queue 是CLH lock的一个变种, 线程节点 node 能否获取lock的判断通过其前继节点
    *      而且这里在当前节点想获取lock时通常给前继节点 打上 signal 的标识(表示前继节点释放lock需要通知我来获取lock)
    *      若这里不清楚的同学, 请先看看 CLH lock的资料 (这是理解 AQS 的基础)
    */
   private Node enq(final Node node){
       for(;;){
           Node t = tail;
           if(t == null){ // Must initialize       // 1. 队列为空 初始化一个 dummy 节点 其实和 ConcurrentLinkedQueue 一样
               if(compareAndSetHead(new Node())){  // 2. 初始化 head 与 tail (这个CAS成功后, head 就有值了, 详情将 Unsafe 操作)
                   tail = head;
               }
           }else{
               node.prev = t;                      // 3. 先设置 Node.pre = pred (PS: 则当一个 node在Sync Queue里面时  node.prev 一定 != null, 但是 node.prev != null 不能说明其在 Sync Queue 里面, 因为现在的CAS可能失败 )
               if(compareAndSetTail(t, node)){     // 4. CAS node 到 tail
                   t.next = node;                  // 5. CAS 成功, 将 pred.next = node (PS: 说明 node.next != null -> 则 node 一定在 Sync Queue, 但若 node 在Sync Queue 里面不一定 node.next != null)
                   return t;
               }
           }
       }
   }
   ~~~

2. Sync Queue节点出队方法：

   ~~~java
   /**
    * 设置 head 节点(在独占模式没有并发的可能, 当共享的模式有可能)
    */
   private void setHead(Node node){
       head = node;
       node.thread = null; // 清除线程引用
       node.prev = null; // 清除原来 head 的引用 <- 都是 help GC
   }
   
   // 清除因中断/超时而放弃获取lock的线程节点(此时节点在 Sync Queue 里面)
   private void cancelAcquire(Node node) {
       if (node == null)
           return;
   
       node.thread = null;                 // 1. 线程引用清空
   
       Node pred = node.prev;
       while (pred.waitStatus > 0)       // 2.  若前继节点是 CANCELLED 的, 则也一并清除
           node.prev = pred = pred.prev;
           
       Node predNext = pred.next;         // 3. 这里的 predNext也是需要清除的(只不过在清除时的 CAS 操作需要 它)
   
       node.waitStatus = Node.CANCELLED; // 4. 标识节点需要清除
   
       // If we are the tail, remove ourselves.
       if (node == tail && compareAndSetTail(node, pred)) { // 5. 若需要清除额节点是尾节点, 则直接 CAS pred为尾节点
           compareAndSetNext(pred, predNext, null);    // 6. 删除节点predNext
       } else {
           int ws;
           if (pred != head &&
                   ((ws = pred.waitStatus) == Node.SIGNAL || // 7. 后继节点需要唤醒(但这里的后继节点predNext已经 CANCELLED 了)
                           (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && // 8. 将 pred 标识为 SIGNAL
                   pred.thread != null) {
               Node next = node.next;
               if (next != null && next.waitStatus <= 0) // 8. next.waitStatus <= 0 表示 next 是个一个想要获取lock的节点
                   compareAndSetNext(pred, predNext, next);
           } else {
               unparkSuccessor(node); // 若 pred 是头节点, 则此刻可能有节点刚刚进入 queue ,所以进行一下唤醒
           }
   
           node.next = node; // help GC
       }
   }
   ~~~

   这里的出Queue的方法其实有两个： 新节点获取lock, 调用setHead抢占head, 并且剔除原head；节点因被中断或获取超时而进行 cancelled, 最后被剔除。

### Condition Queue

Condition Queue 是一个并发不安全的，只用于独占模式的队列（PS: 为什么是并发不安全的呢? 主要是在操作 Condition 时，线程必需获取 独占的 lock，所以不需要考虑并发的安全问题）， 而当Node存在于 Condition Queue 里面，则其只有 waitStatus，thread，nextWaiter 有值，其他的都是null（其中的 waitStatus 只能是 CONDITION，0 代表node进行转移到 Sync Queue里面，或被中断/timeout），这里有个注意点，就是当线程被中断或获取 lock 超时，则一瞬间 node 会存在于 Condition Queue，Sync Queue 两个队列中。

![](aqs\aqs04.png)

节点 Node4, Node5, Node6, Node7 都是调用 Condition.awaitXX 方法加入 Condition Queue(PS: 加入后会将原来的 lock 释放)。

1. 入队方法addConditionWaiter

   将当前线程封装成一个 Node 节点放入到 Condition Queue 里面大家可以注意到，下面对 Condition Queue 的操作都没考虑到并发（Sync Queue 的队列是支持并发操作的），这是为什么呢？因为在进行操作 Condition 是当前的线程已经获取了AQS的独占锁，所以不需要考虑并发的情况。

   ~~~java
   private Node addConditionWaiter(){
       Node t = lastWaiter;                                
       // Condition queue 的尾节点           
   	// 尾节点已经Cancel, 直接进行清除,
       /** 
       * 当Condition进行 awiat 超时或被中断时, Condition里面的节点是没有被删除掉的, 需要其	 * 他await 在将线程加入 Condition Queue 时调用addConditionWaiter而进而删除, 或 await 操作差不多结束时, 调用 "node.nextWaiter != null" 进行判断而删除 (PS: 通过 signal 进行唤
       * 醒时 node.nextWaiter 会被置空, 而中断和超时时不会)
       */
       if(t != null && t.waitStatus != Node.CONDITION){
       	/** 
       	* 调用 unlinkCancelledWaiters 对 "waitStatus != Node.CONDITION" 的节点进行		* 删除(在Condition里面的Node的waitStatus 要么是CONDITION(正常), 要么就是 0 
       	* (signal/timeout/interrupt))
       	*/
           unlinkCancelledWaiters();                     
           t = lastWaiter;                     
       }
       //将线程封装成 node 准备放入 Condition Queue 里面
       Node node = new Node(Thread.currentThread(), Node.CONDITION);
       if(t == null){
       	//Condition Queue 是空的
           firstWaiter = node;                           
       } else {
       	// 追加到 queue 尾部
           t.nextWaiter = node;                          
       }
       lastWaiter = node;                               
       return node;
   }
   ~~~

2. 删除Cancelled节点的方法unlinkCancelledWaiters

   当Node在Condition Queue 中，若状态不是 CONDITION，则一定是被中断或超时。在调用 addConditionWaiter 将线程放入Condition Queue 里面时或 awiat 方法获取结束时进行清理 Condition queue 里面的因 timeout/interrupt 而还存在的节点。这个删除操作比较巧妙，其中引入了 trail 节点， 可以理解为traverse整个 Condition Queue 时遇到的最后一个有效的节点。

   ~~~java
   private void unlinkCancelledWaiters(){
       Node t = firstWaiter;
       Node trail = null;
       while(t != null){
           Node next = t.nextWaiter;               // 1. 先初始化 next 节点
           if(t.waitStatus != Node.CONDITION){   // 2. 节点不有效, 在Condition Queue 里面 Node.waitStatus 只有可能是 CONDITION 或是 0(timeout/interrupt引起的)
               t.nextWaiter = null;               // 3. Node.nextWaiter 置空
               if(trail == null){                  // 4. 一次都没有遇到有效的节点
                   firstWaiter = next;            // 5. 将 next 赋值给 firstWaiter(此时 next 可能也是无效的, 这只是一个临时处理)
               } else {
                   trail.nextWaiter = next;       // 6. next 赋值给 trail.nextWaiter, 这一步其实就是删除节点 t
               }
               if(next == null){                  // 7. next == null 说明 已经 traverse 完了 Condition Queue
                   lastWaiter = trail;
               }
           }else{
               trail = t;                         // 8. 将有效节点赋值给 trail
           }
           t = next;
       }
   }
   ~~~

3. 转移节点的方法transferForSignal

   transferForSignal只有在节点被正常唤醒才调用的正常转移的方法。将Node 从Condition Queue 转移到 Sync Queue 里面在调用transferForSignal之前，会 first.nextWaiter = null，而我们发现若节点是因为 timeout / interrupt 进行转移，则不会进行这步操作，两种情况的转移都会把 wautStatus 置为 0。

   ~~~java
   final boolean transferForSignal(Node node){
       /**
        * If cannot change waitStatus, the node has been cancelled
        */
       if(!compareAndSetWaitStatus(node, Node.CONDITION, 0)){ // 1. 若 node 已经 cancelled 则失败
           return false;
       }
   
       Node p = enq(node);                                 // 2. 加入 Sync Queue
       int ws = p.waitStatus;
       if(ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)){ // 3. 这里的 ws > 0 指Sync Queue 中node 的前继节点cancelled 了, 所以, 唤醒一下 node ; compareAndSetWaitStatus(p, ws, Node.SIGNAL)失败, 则说明 前继节点已经变成 SIGNAL 或 cancelled, 所以也要 唤醒
           LockSupport.unpark(node.thread);
       }
       return true;
   }
   ~~~

4. 转移节点的方法transferAfterCancelledWait 

   transferAfterCancelledWait 在节点获取lock时被中断或获取超时才调用的转移方法。将 Condition Queue 中因 timeout/interrupt 而唤醒的节点进行转移。

   ~~~java
   final boolean transferAfterCancelledWait(Node node){
       if(compareAndSetWaitStatus(node, Node.CONDITION, 0)){ // 1. 没有 node 没有 cancelled , 直接进行转移 (转移后, Sync Queue , Condition Queue 都会存在 node)
           enq(node);
           return true;
       }
       
       while(!isOnSyncQueue(node)){                // 2.这时是其他的线程发送signal,将本线程转移到 Sync Queue 里面的工程中(转移的过程中 waitStatus = 0了, 所以上面的 CAS 操作失败)
           Thread.yield();                         // 这里调用 isOnSyncQueue判断是否已经 入Sync Queue 了
       }
       return false;
   }
   ~~~

### 独占式

独占式获取和释放资源使用的方法为：

~~~java
void acquire(int arg)
void acquireInterruptibly(int arg) //对中断响应
boolean release(int arg)
~~~

独占式获取的资源是与具体线程绑定的，如获取到ReentrantLock的锁后，在AQS内部会先使用CAS操作把state状态值从0变为1，然后设置当前锁的持有者为当前线程，其他线程则会被放入AQS阻塞队列后挂起。

#### 获取资源acquire

~~~java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
~~~

首先使用tryAcquire方法尝试获取资源，具体是设置状态变量state的值，成功则直接返回，失败则将当前线程封装为类型为Node.EXCLUSIVE的Node节点后插入到AQS阻塞队列的尾部，并调用LockSupport.park(this)方法挂起自己。

函数流程如下：

- tryAcquire()尝试直接去获取资源，如果成功则直接返回；
- addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；
- acquireQueued()使线程在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。
- 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

1. 将当前线程加入到队列队尾addWaiter

   ~~~java
   private Node addWaiter(Node mode) {
       //以给定模式构造结点。mode有两种：EXCLUSIVE（独占）和SHARED（共享）
       Node node = new Node(Thread.currentThread(), mode);
       //尝试快速方式直接放到队尾
       Node pred = tail;
       //如果尾结点不为空，CAS快速尝试在尾部添加，若CAS设置成功，返回；否则，eng
       if (pred != null) {
           node.prev = pred;
           if (compareAndSetTail(pred, node)) {
               pred.next = node;
               return node;
           }
       }
       //上一步失败则通过enq入队
       enq(node);
       return node;
   }
   ~~~

   Node结点是对每一个访问同步代码的线程的封装，其包含了需要同步的线程本身以及线程的状态，如是否被阻塞，是否等待唤醒，是否已经被取消等。变量waitStatus则表示当前被封装成Node结点的等待状态，共有4种取值CANCELLED、SIGNAL、CONDITION、PROPAGATE。

   - CANCELLED：值为1，在同步队列中等待的线程等待超时或被中断，需要从同步队列中取消该Node的结点，即结束状态，进入该状态后的结点将不会再变化。
   - SIGNAL：值为-1，被标识为该等待唤醒状态的后继结点，当其前继结点的线程释放了同步锁或被取消，将会通知该后继结点的线程执行。其就是处于唤醒状态，只要前继结点释放锁，就会通知标识为SIGNAL状态的后继结点的线程执行。
   - CONDITION：值为-2，与Condition相关，该标识的结点处于等待队列中，当其他线程调用了Condition的signal()方法后，CONDITION状态的结点将从等待队列转移到同步队列中，等待获取同步锁。
   - PROPAGATE：值为-3，与共享模式相关，在共享模式中，该状态标识结点的线程处于可运行状态。
   - 0状态：值为0，代表初始化状态。

   AQS在判断状态时，通过用waitStatus>0表示取消状态，而waitStatus<0表示有效状态。

2. 将node加入队尾enq

   ~~~java
   private Node enq(final Node node) {
       //CAS"自旋"，直到成功加入队尾
       for (;;) {
           Node t = tail;
           // 队列为空，创建一个空的标志结点作为head结点，并将tail也指向它。
           if (t == null) { 
               if (compareAndSetHead(new Node()))
                   tail = head;
           } else {
               node.prev = t;
               //cas设置尾结点，不成功就一直重试
               if (compareAndSetTail(t, node)) {
                   t.next = node;
                   return t;
               }
           }
       }
   }
   ~~~

   这段代码的精华就是**CAS自旋volatile变量**，是一种很经典的用法。

3. 在等待队列中排队拿号acquireQueued

   ~~~java
   final boolean acquireQueued(final Node node, int arg) {
       //标记是否成功拿到资源
       boolean failed = true;
       try {
           //标记等待过程中是否被中断过
           boolean interrupted = false;
           //CAS自旋
           for (;;) {
               //拿到前驱结点
               final Node p = node.predecessor();
               //如果前驱是head，即该结点已成老二，那么便有资格去尝试获取资源（可能是老大释放完资源唤醒自己的，当然也可能被interrupt了）
               if (p == head && tryAcquire(arg)) {
                   //拿到资源后，将head指向该结点。所以head所指的标杆结点，就是当前获取到资源的那个结点或null
                   setHead(node);
                   // setHead中node.prev已置为null，此处再将head.next置为null，就是为了方便GC回收以前的head结点。也就意味着之前拿完资源的结点出队了
                   p.next = null; // help GC
                   failed = false;
                   //返回等待过程中是否被中断过
                   return interrupted;
               }
               // 如果没有获取到同步状态，通过shouldParkAfterFailedAcquire判断是否应该阻塞，parkAndCheckInterrupt用来阻塞线程
               if (shouldParkAfterFailedAcquire(p, node) &&
                   parkAndCheckInterrupt())
                   //如果等待过程中被中断过，哪怕只有那么一次，就将interrupted标记为true
                   interrupted = true;
           }
       } finally {
           if (failed)
               cancelAcquire(node);
       }
   }
   ~~~

   该函数的具体流程：

   - 结点进入队尾后，检查状态，找到安全休息点；
   - 调用park()进入waiting状态，等待unpark()或interrupt()唤醒自己；
   - 被唤醒后，看自己是不是有资格能拿到号。如果拿到，head指向当前结点，并返回从入队到拿到号的整个过程中是否被中断过；如果没拿到，继续流程1。

4. 进入waiting状态shouldParkAfterFailedAcquire

   ~~~java
   private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
       //拿到前驱的状态
       int ws = pred.waitStatus;
       if (ws == Node.SIGNAL)
           //如果已经告诉前驱拿完号后通知自己一下，那就可以安心休息了
           return true;
       if (ws > 0) {
          /*
           * 如果前驱放弃了，那就一直往前找，直到找到最近一个正常等待的状态，并排在它的后边。
           * 注：那些放弃的结点，由于被自己“加塞”到它们前边，它们相当于形成一个无引用链，稍后就会被GC
           */
           do {
               node.prev = pred = pred.prev;
           } while (pred.waitStatus > 0);
           pred.next = node;
       } else {
           //如果前驱正常，那就把前驱的状态设置成SIGNAL，告诉它拿完号后通知自己一下。有可能失败，人家说不定刚刚释放完呢！
           compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
       }
       return false;
   }
   ~~~

   整个流程中，如果前驱结点的状态不是SIGNAL，那么自己就需要去找个安心的休息点，同时可以再尝试下看有没有机会轮到自己拿号。

5. 让线程真正进入等待状态parkAndCheckInterrupt

   ~~~java
   private final boolean parkAndCheckInterrupt() {
       //调用park()使线程进入waiting状态
       LockSupport.park(this);
       //如果被唤醒，查看自己是不是被中断的。
       return Thread.interrupted();
   }
   ~~~

   park()会让当前线程进入waiting状态。在此状态下，有两种途径可以唤醒该线程：

   - 被unpark()
   - 被interrupt()

   需要注意的是，Thread.interrupted()会清除当前线程的中断标记位。 

acquire流程图总结：

![](aqs\aqs02.png)



#### 释放资源release

~~~java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            //唤醒等待队列里的下一个线程
            unparkSuccessor(h);
        return true;
    }
    return false;
}
~~~

首先使用tryRelease方法释放资源，这里是设置状态变量state的值，然后调用LockSupport.unpark(thread)方法激活AQS队列里面被阻塞的一个线程。被激活的线程则使用tryAcquire尝试获取资源，满足则该线程被激活，不满足还是会被放入AQS队列并被挂起。

release()是根据tryRelease()的返回值来判断该线程是否已经完成释放掉资源了！所以自义定同步器在实现时，如果已经彻底释放资源(state=0)，要返回true，否则返回false。

~~~java
private void unparkSuccessor(Node node) {
    //获取当前线程所在的结点状态
    int ws = node.waitStatus;
    if (ws < 0)
        //将等待状态waitStatus设置为初始值0，允许失败
        compareAndSetWaitStatus(node, ws, 0);

    //找到下一个需要唤醒的结点s
    Node s = node.next;
    //如果为空或已取消，则从后尾部往前遍历找到一个处于正常阻塞状态的结点进行唤醒
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            //从这里可以看出，<=0的结点，都是还有效的结点。
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
~~~

**用unpark()唤醒等待队列中最前边的那个未放弃线程**，这里用s来表示。此时，再和acquireQueued()联系起来，s被唤醒后，进入if (p == head && tryAcquire(arg))的判断，即使p!=head也没关系，它会再进入shouldParkAfterFailedAcquire()寻找一个安全点。这里既然s已经是等待队列中最前边的那个未放弃线程了，那么通过shouldParkAfterFailedAcquire()的调整，s也必然会跑到head的next结点，下一次自旋p==head就成立了，然后s把自己设置成head标杆结点，表示自己已经获取到资源了，acquire()也返回了。

注：AQS是锁阻塞和同步器的基础框架，所以并没有提供tryAcquire和tryRelease方法，需要由具体子类来实现，子类要根据具体场景使用CAS算法尝试修改state状态值，还需要定义在调用acquire和release方法时state状态值的增减代表的含义。如：ReentrantLock：

- 重写tryAcquire时，使用CAS算法查看当前state是否为0，为0则使用CAS设置为1，并设置当前锁的持有者为当前线程，并返回true，失败则返回false；
- 重写tryRelease时，使用CAS算法把当前state的值从1修改为0，并设置当前锁的持有者为null，并返回true，失败则返回false

### 共享式

共享式获取和释放资源的方法为：

~~~java
void acquireShared(int arg)
void acquireSharedInterruptibly(int arg) //对中断响应
boolean releaseShared(int arg)
~~~

共享式获取的资源与具体线程是不相关的，当多个线程通过CAS方式竞争获取资源，当一个线程获取到了资源后，另一个线程会再次获取，满足则使用CAS方式获取即可，不满足则把当前线程放入阻塞队列。

#### 获取资源acquireShared

~~~java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
~~~

首先使用tryAcquireShared尝试获取资源，具体是设置状态变量state的值，成功则直接返回，失败则将当前线程封装为Node.SHARED的Node节点后插入到AQS阻塞队列的尾部，并调用LockSupport.park(this)方法挂起自己。

AQS已经把tryAcquireShared返回值的语义定义好了：

- 负值代表获取失败；
- 0代表获取成功，但没有剩余资源；
- 正数表示获取成功，还有剩余资源，其他线程还可以去获取。

所以acquireShared()的流程就是：

- tryAcquireShared()尝试获取资源，成功则直接返回；
- 失败则通过doAcquireShared()进入等待队列park()，直到被unpark()/interrupt()并成功获取到资源才返回。整个等待过程也是忽略中断的

1. 将当前线程加入等待队列尾部休息doAcquireShared

   ~~~java
   private void doAcquireShared(int arg) {
       //加入队列尾部
       final Node node = addWaiter(Node.SHARED);
       //是否成功标志
       boolean failed = true;
       try {
           //等待过程中是否被中断过的标志
           boolean interrupted = false;
           for (;;) {
               //前驱结点
               final Node p = node.predecessor();
               //如果到head的下一个，因为head是拿到资源的线程，此时node被唤醒，很可能是head用完资源来唤醒自己的
               if (p == head) {
                   //尝试获取资源
                   int r = tryAcquireShared(arg);
                   if (r >= 0) {//成功
                       //将head指向自己，还有剩余资源可以再唤醒之后的线程
                       setHeadAndPropagate(node, r);
                       p.next = null; // help GC
                       //如果等待过程中被打断过，此时将中断补上。
                       if (interrupted)
                           selfInterrupt();
                       failed = false;
                       return;
                   }
               }
               //判断状态，寻找安全点，进入waiting状态，等着被unpark()或interrupt()
               if (shouldParkAfterFailedAcquire(p, node) &&
                   parkAndCheckInterrupt())
                   interrupted = true;
           }
       } finally {
           if (failed)
               cancelAcquire(node);
       }
   }
   ~~~

   与acquireQueued()相比，只不过这里将补中断的selfInterrupt()放到doAcquireShared()里了，而独占模式是放到acquireQueued()之外，其实都一样。

   跟独占模式比，还有一点需要注意的是，这里只有线程是head.next时，老二，才会去尝试获取资源，有剩余的话还会唤醒之后的队友。那么问题就来了，假如老大用完后释放了5个资源，而老二需要6个，老三需要1个，老四需要2个。老大先唤醒老二，老二一看资源不够，他是把资源让给老三呢，还是不让？答案是否定的！老二会继续park()等待其他线程释放资源，也更不会去唤醒老三和老四了。独占模式，同一时刻只有一个线程去执行，这样做未尝不可；但共享模式下，多个线程是可以同时执行的，现在因为老二的资源需求量大，而把后面量小的老三和老四也都卡住了。当然，这并不是问题，只是AQS保证严格按照入队顺序唤醒罢了（保证公平，但降低了并发）。

2. setHeadAndPropagate

   ~~~java
   private void setHeadAndPropagate(Node node, int propagate) {
       Node h = head; 
       //head指向自己
       setHead(node);
   	//如果还有剩余量，继续唤醒下一个邻居线程
       if (propagate > 0 || h == null || h.waitStatus < 0 ||
           (h = head) == null || h.waitStatus < 0) {
           Node s = node.next;
           if (s == null || s.isShared())
               doReleaseShared();
       }
   }
   ~~~

#### 释放资源releaseShared

~~~java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
~~~

首先使用tryReleaseShared方法释放资源，这里是设置状态变量state的值，然后调用LockSupport.unpark(thread)方法激活AQS队列里面被阻塞的一个线程。被激活的线程则使用tryAcquireShared尝试获取资源，满足则该线程被激活，不满足还是会被放入AQS队列并被挂起。

一句话：释放掉资源后，唤醒后继。跟独占模式下的release()相似，区别是：

- 独占模式下的tryRelease()在完全释放掉资源（state=0）后，才会返回true去唤醒其他线程，这主要是基于独占下可重入的考量；
- 而共享模式下的releaseShared()则没有这种要求，共享模式实质就是控制一定量的线程并发执行，那么拥有资源的线程在释放掉部分资源时就可以唤醒后继等待结点。例如，资源总量是13，A（5）和B（7）分别获取到资源并发运行，C（4）来时只剩1个资源就需要等待。A在运行过程中释放掉2个资源量，然后tryReleaseShared(2)返回true唤醒C，C一看只有3个仍不够继续等待；随后B又释放2个，tryReleaseShared(2)返回true唤醒C，C一看有5个够自己用了，然后C就可以跟A和B一起运行。而ReentrantReadWriteLock读锁的tryReleaseShared()只有在完全释放掉资源（state=0）才返回true，所以自定义同步器可以根据需要决定tryReleaseShared()的返回值。 

~~~java
private void doReleaseShared() {
    //死循环，共享模式，持有同步状态的线程可能有多个，采用循环CAS保证线程安全
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                //唤醒后继
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
~~~

注：同样，AQS也没有提供tryAcquireShared、tryReleaseShared。

基于AQS实现的锁除了需要重写上述方法外，还需要重写isHeldExclusively方法，来判断锁是被当前线程独占还是被共享。

### 设计思想

对于使用者来讲，我们无需关心获取资源失败，线程排队，线程阻塞/唤醒等一系列复杂的实现，这些都在AQS中为我们处理好了。我们只需要负责好自己的那个环节就好，也就是获取/释放共享资源state的姿势。很经典的模板方法设计模式的应用，AQS为我们定义好顶级逻辑的骨架，并提取出公用的线程入队列/出队列，阻塞/唤醒等一系列复杂逻辑的实现，将部分简单的可由使用者决定的操作逻辑延迟到子类中去实现即可。

### 自定义同步锁

自定义同步器在实现时只需要实现共享资源state的获取与释放方式即可，至于具体线程等待队列的维护（如获取资源失败入队/唤醒出队等），AQS已经在顶层实现好了。

自定义同步器实现时主要实现以下几种方法：

1. isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它；
2. tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false；
3. tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false；
4. tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源；
5. tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

以ReentrantLock为例，state初始化为0，表示未锁定状态。A线程lock()时，会调用tryAcquire()独占该锁并将state+1。此后，其他线程再tryAcquire()时就会失败，直到A线程unlock()到state=0，即释放锁为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的，state会累加，这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证state是能回到零态的。

再以CountDownLatch以例，任务分为N个子线程去执行，state也初始化为N，注意N要与线程个数一致。这N个子线程是并行执行的，每个子线程执行完后countDown()一次，state会CAS减1。等到所有子线程都执行完后，即state=0，会unpark()主调用线程，然后主调用线程就会从await()函数返回，继续后余动作。

一般来说，自定义同步器要么是独占方法，要么是共享方式，他们也只需实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared中的一种即可。但AQS也支持自定义同步器同时实现独占和共享两种方式，如ReentrantReadWriteLock。

Mutex是一个不可重入的互斥锁实现。锁资源（AQS里的state）只有两种状态：0表示未锁定，1表示锁定。下边是Mutex的核心源码：

~~~java
class Mutex implements Lock, java.io.Serializable {
    // 自定义同步器
    private static class Sync extends AbstractQueuedSynchronizer {
        // 判断是否锁定状态
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // 尝试获取资源，立即返回。成功则返回true，否则false。
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // 这里限定只能为1个量
            if (compareAndSetState(0, 1)) {//state为0才设置为1，不可重入！
                setExclusiveOwnerThread(Thread.currentThread());//设置为当前线程独占资源
                return true;
            }
            return false;
        }

        // 尝试释放资源，立即返回。成功则为true，否则false。
        protected boolean tryRelease(int releases) {
            assert releases == 1; // 限定为1个量
            if (getState() == 0)//既然来释放，那肯定就是已占有状态了。只是为了保险，多层判断！
                throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);//释放资源，放弃占有状态
            return true;
        }
    }

    // 真正同步类的实现都依赖继承于AQS的自定义同步器！
    private final Sync sync = new Sync();

    //lock<-->acquire。两者语义一样：获取资源，即便等待，直到成功才返回。
    public void lock() {
        sync.acquire(1);
    }

    //tryLock<-->tryAcquire。两者语义一样：尝试获取资源，要求立即返回。成功则为true，失败则为false。
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    //unlock<-->release。两者语文一样：释放资源。
    public void unlock() {
        sync.release(1);
    }

    //锁是否占有状态
    public boolean isLocked() {
        return sync.isHeldExclusively();
    }
}
~~~

同步类在实现时一般都将自定义同步器（sync）定义为内部类，供自己使用；而同步类自己（Mutex）则实现某个接口，对外服务。当然，接口的实现要直接依赖sync，它们在语义上也存在某种对应关系！！而sync只用实现资源state的获取-释放方式tryAcquire-tryRelelase，至于线程的排队、等待、唤醒等，上层的AQS都已经实现好了，我们不用关心。

### 参考资料

1. [java并发之AQS详解](https://www.cnblogs.com/waterystone/p/4920797.html)
2. [java并发包基石之AQS详解](https://www.cnblogs.com/chengxiao/p/7141160.html)
3. [并发Lock之AQS](https://juejin.im/post/5a4a4530518825697078553e)