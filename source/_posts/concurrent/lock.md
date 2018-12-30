---
title: java并发包锁原理解析（一）：Lock
date: 2018-12-30 12:47:24
tags:
 - 并发包
 - 锁
 - 面试
categories: 
 - 并发包
---

### 介绍

Java 5中引入了新的锁机制—java.util.concurrent.locks中的显式的互斥锁：**Lock接口**，它提供了比synchronized更加广泛的锁定操作。

Lock接口实现类：ReentrantLock可重入独占锁。

lock必须被显式地创建、锁定和释放，为了可以使用更多的功能，一般用ReentrantLock为其实例化。

为了保证锁最终一定会被释放（可能会有异常发生），要把互斥区放在try语句块内，并在finally语句块中释放锁，尤其当有return语句时，return语句必须放在try字句中，以确保unlock（）不会过早发生，从而将数据暴露给第二个任务。因此，采用lock加锁和释放锁的一般形式如下：

~~~java
//默认使用非公平锁，如果要使用公平锁，需要传入参数true
Lock lock = new ReentrantLock();
lock.lock();
try {
    //更新对象的状态
    //捕获异常，必要时恢复到原来的不变约束
    //如果有return语句，放在这里
} finally {
    //锁必须在finally块中释放
    lock.unlock();        
}
~~~

<!-- more -->

### synchronized的缺陷

synchronized是java中的一个关键字，也就是说是Java语言内置的特性。

1. 释放锁情况

   被synchronized修饰的代码块，当一个线程获取了对应的锁，并执行该代码块时，其他线程便只能一直等待，等待获取锁的线程释放锁，而这里获取锁的线程释放锁只会有两种情况：

   - 获取锁的线程执行完了该代码块，然后线程释放对锁的占有；
   - 线程执行发生异常，此时JVM会让线程自动释放锁。

   如果这个获取锁的线程由于要等待IO或者其他原因（比如调用sleep方法）被阻塞了，但是又没有释放锁，其他线程便只能干巴巴地等待。

2. 读写锁情况

   当有多个线程读写文件时，读操作和写操作会发生冲突现象，写操作和写操作会发生冲突现象，但是读操作和读操作不会发生冲突现象。但是采用synchronized关键字来实现同步的话，就会导致一个问题：如果多个线程都只是进行读操作，所以当一个线程在进行读操作时，其他线程只能等待无法进行读操作。

   因此就需要一种机制来使得多个线程都只是进行读操作时，线程之间不会发生冲突，通过Lock就可以办到。

3. 通过Lock可以知道线程有没有成功获取到锁。这个是synchronized无法办到的

Lock和synchronized有一点非常大的不同，采用synchronized不需要用户去手动释放锁，当synchronized方法或者synchronized代码块执行完之后，系统会自动让线程释放对锁的占用；而Lock则必须要用户去手动释放锁，如果没有主动释放锁，就有可能导致出现死锁现象。

### Lock接口

Lock是一个接口：

~~~java
public interface Lock {
    void lock();
    // 响应中断
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
~~~

1. lock

   lock方法用来获取锁，如果锁已被其他线程获取，则进行等待。

   采用Lock，必须主动去释放锁，并且在发生异常时，不会自动释放锁。因此使用Lock必须在try{}catch{}块中进行，并且将释放锁的操作放在finally块中进行，以保证锁一定被被释放，防止死锁的发生。

2. tryLock

   tryLock方法是有返回值的，它表示用来尝试获取锁，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false，也就说这个方法无论如何都会立即返回。

3. lockInterruptibly

   lockInterruptibly方法比较特殊，当通过这个方法去获取锁时，如果线程正在等待获取锁，则这个线程能够响应中断，即中断线程的等待状态。也就使说，当两个线程同时通过lock.lockInterruptibly()想获取某个锁时，假若此时线程A获取到了锁，而线程B只有在等待，那么对线程B调用threadB.interrupt()方法能够中断线程B的等待过程。

   注意：当一个线程获取了锁之后，是不会被interrupt方法中断的，只能中断阻塞过程中的线程。

4. newCondition

   条件变量或条件队列，可多次调用，可以通过绑定Condition对象来判断当前线程通知的是哪些线程（即与Condition对象绑定在一起的其他线程）。 

### ReentrantLock

ReentrantLock是唯一实现了Lock接口的类，并且ReentrantLock提供了更多的方法。ReentrantLock最终还是使用AQS实现，并且根据参数来决定其内部是个公平锁还是非公平锁，默认非公平锁：

~~~java
public ReentrantLock() {
    sync = new NonfairSync();
}
~~~

~~~java
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
~~~

在这里，AQS的state状态值表示线程获取锁的可重入次数，默认情况下，state的值为0表示当前锁没有被任何线程持有。

~~~java
abstract static class Sync extends AbstractQueuedSynchronizer {}
~~~

Sync类继承自AQS，它的子类FairSync和NonfairSync分别实现了获取锁的公平和非公平策略。

#### FairSync

~~~java
static final class FairSync extends Sync {
    final void lock() {
        // 调用AQS的acquire方法
        acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 当前AQS状态值为0
        if (c == 0) {
            // 公平性策略
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 当前线程是该锁持有者
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
~~~

AQS的acquire方法：

~~~java
public final void acquire(int arg) {
    // 调用ReentrantLock重写的tryAcquire方法
    if (!tryAcquire(arg) &&
        // tryAcquiref返回false会把当前线程放入到AQS阻塞队列
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
~~~

公平锁与飞公平锁的区别就在于hasQueuedPredecessors方法：

~~~java
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
~~~

如果当前线程节点有前驱节点则返回true，否则如果当前AQS队列为空或者当前线程节点是AQS的第一个节点则返回false。其中如果h==t则说明当前队列为空，直接返回false；如果h!=t并且s==null则说明有一个元素将要作为AQS的第一个节点入队列（enq函数的第一个元素入队列是两步操作：首先创建第一个哨兵头节点，然后将第一个元素插入到哨兵节点后面），那么返回true；如果h!=t并且s!=null和s.thread != Thread.currentThread()则说明队列里面的第一个元素不是当前线程，那么返回true。

####NonfairSync

~~~java
static final class NonfairSync extends Sync {
    final void lock() {
        // CAS设置状态值
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 调用AQS的acquire方法
            acquire(1);
    }

    /**
     * ReentrantLock重写的tryAcquire方法
     */
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
~~~

nonfairTryAcquire方法：

~~~java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 当前AQS状态值为0
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 当前线程是该锁的持有者
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
~~~

#### 获取锁

1. void lock

   ~~~java
   public void lock() {
       sync.lock();
   }
   ~~~

   ReetrankLock的lock方法委托给了sync类，根据创建ReetrankLock构造函数选择sync的实现是FairSync还是NonfairSync。

2. void lockInterruptibly

   与lock方法类似，它对中断进行响应，就是当前线程在调用该方法时，如果其他线程调用了当前线程的interrupt方法，则当前线程会抛出异常，然后返回：

   ~~~java
   public void lockInterruptibly() throws InterruptedException {
       // 调用AQSacquireInterruptibly方法
       sync.acquireInterruptibly(1);
   }
   ~~~

   ~~~java
   public final void acquireInterruptibly(int arg)
               throws InterruptedException {
       // 如果当前线程被中断，则直接抛出异常
       if (Thread.interrupted())
           throw new InterruptedException();
       // 尝试获取资源
       if (!tryAcquire(arg))
           // 调用AQS可被中断的方法
           doAcquireInterruptibly(arg);
   }
   ~~~

3. boolean tryLock

   尝试获取锁，如果当前该锁没有被其他线程持有，则当前线程获取该锁并返回true。该方法不会引起当前线程阻塞：

   ~~~java
   public boolean tryLock() {
       return sync.nonfairTryAcquire(1);
   }
   ~~~

   与非公平锁的tryAcquire方法代码类似，所以tryLock使用的是非公平策略。

4. boolean tryLock(long timeout, TimeUnit unit)

   与tryLock的不同之处是设置了超时时间，如果超时时间到了没有获取到该锁则返回false：

   ~~~java
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
       // 调用AQS的tryAcquireNanos方法
       return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
   ~~~

#### 释放锁

调用该方法使AQS状态值减1，如果减去1后为0则释放该锁。如果当前线程没有持有该锁，调用该方法会抛出IllegalMonitorStateException异常：

~~~java
public void unlock() {
    // 调用AQSrelease方法
    sync.release(1);
}
~~~

~~~java
protected final boolean tryRelease(int releases) {
    // 如果不是锁持有者调用Unlock则抛出异常
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 如果当前可重入次数为0，则清空锁持有线程
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    // 设置可重入锁次数为原始值-1
    setState(c);
    return free;
}
~~~

#### 条件变量

![](lock\ReetrankLock01.png)

如上图：假如线程Thread1、Thread2、Thread3同时尝试获取独占锁ReetrankLock，假设Thread1获取到了，则Thread2、Thread3就会被转换为Node节点并被放入ReetrankLock对应的AQS阻塞队列，而后被阻塞挂起。

假设Thread1获取锁后调用了对应的锁创建的条件变量1，那么Thread1就会释放获取到的锁，然后当前线程就会被转换为Node节点插入条件变量1的条件队列。由于Thread1释放了锁，所以阻塞到AQS队列里面的Thread2和Thread3就有机会获取到锁，假设使用的公平策略，那么Thread2会获取到该锁，从而从AQS队列里面移除Thread2对应得Node节点。如下图：

![](lock\ReetrankLock02.png)

### ReetrankLock与synchronized比较

#### 性能比较

在JDK1.5中，synchronized是性能低效的。因为这是一个重量级操作，它对性能最大的影响是阻塞的是实现，挂起线程和恢复线程的操作都需要转入内核态中完成，这些操作给系统的并发性带来了很大的压力。相比之下使用Java提供的Lock对象，性能更高一些。这两种锁在JDK1.5、单核处理器及双Xeon处理器环境下做了一组吞吐量对比的实验，发现多线程环境下，synchronized的吞吐量下降的非常严重，而ReentrankLock则能基本保持在同一个比较稳定的水平上。

到了JDK1.6，发生了变化，对synchronize加入了很多优化措施，有自适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等。导致在JDK1.6上synchronize的性能并不比Lock差。官方也表示，他们也更支持synchronize，在未来的版本中还有优化余地，所以还是提倡在synchronized能实现需求的情况下，优先考虑使用synchronized来进行同步。

两种锁机制的底层的实现策略：

1. synchronized采用的并发策略

   互斥同步最主要的问题就是进行线程阻塞和唤醒所带来的性能问题，因而这种同步又称为阻塞同步，它属于一种悲观的并发策略，即线程获得的是独占锁。独占锁意味着其他线程只能依靠阻塞来等待线程释放锁。而在CPU转换线程阻塞时会引起线程上下文切换，当有很多线程竞争锁的时候，会引起CPU频繁的上下文切换导致效率很低。

2. ReetrantLock采用的并发策略

   随着指令集的发展，我们有了另一种选择：基于冲突检测的乐观并发策略，通俗地讲就是先进性操作，如果没有其他线程争用共享数据，那操作就成功了，如果共享数据被争用，产生了冲突，那就再进行其他的补偿措施（最常见的补偿措施就是不断地重拾，直到试成功为止），这种乐观的并发策略的许多实现都不需要把线程挂起，因此这种同步被称为非阻塞同步。

在乐观的并发策略中，需要操作和冲突检测这两个步骤具备原子性，它靠硬件指令来保证，这里用的是CAS操作（Compare and Swap）。JDK1.5之后，Java程序才可以使用CAS操作。现代的CPU提供了指令，可以自动更新共享数据，而且能够检测到其他线程的干扰，而compareAndSet() 就用这些代替了锁定。这个算法称作非阻塞算法，意思是一个线程的失败或者挂起不应该影响其他线程的失败或挂起。

Java 5中引入了注入AutomicInteger、AutomicLong、AutomicReference等特殊的原子性变量类，它们提供的如：compareAndSet（）、incrementAndSet（）和getAndIncrement（）等方法都使用了CAS操作。因此，它们都是由硬件指令来保证的原子方法。

#### 用途比较

基本语法上，ReentrantLock与synchronized很相似，它们都具备一样的线程重入特性，只是代码写法上有点区别而已，一个表现为API层面的互斥锁Lock，一个表现为原生语法层面的互斥锁synchronized。ReentrantLock相对synchronized而言还是增加了一些高级功能，主要有以下三项：

1. 等待可中断：当持有锁的线程长期不释放锁时，正在等待的线程可以选择放弃等待，改为处理其他事情，它对处理执行时间非常上的同步块很有帮助。而在等待由synchronized产生的互斥锁时，会一直阻塞，是不能被中断的。
2. 可实现公平锁：多个线程在等待同一个锁时，必须按照申请锁的时间顺序排队等待，而非公平锁则不保证这点，在锁释放时，任何一个等待锁的线程都有机会获得锁。synchronized中的锁时非公平锁，ReentrantLock默认情况下也是非公平锁，但可以通过构造方法ReentrantLock(ture)来要求使用公平锁。
3. 锁可以绑定多个条件：ReentrantLock对象可以同时绑定多个Condition对象（条件变量或条件队列），而在synchronized中，锁对象的wait和notify或notifyAll方法可以实现一个隐含条件，但如果要和多于一个的条件关联的时候，就不得不额外地添加一个锁，而ReentrantLock则无需这么做，只需要多次调用newCondition方法即可。而且我们还可以通过绑定Condition对象来判断当前线程通知的是哪些线程（即与Condition对象绑定在一起的其他线程）。

### 条件变量实现线程协作

在生产者—消费者模型中，可以用synchronized实现互斥，并配合使用Object对象的wait和notify或notifyAll方法来实现线程间协作。

Java 5之后，可以用Reentrantlock锁配合Condition对象上的await和signal或signalAll方法来实现线程间协作。在ReentrantLock对象上newCondition可以得到一个Condition对象，可以通过在Condition上调用await方法来挂起一个任务，通过在Condition上调用signal来通知任务，从而唤醒一个任务，或者调用signalAll来唤醒所有在这个Condition上被其自身挂起的任务。

另外，如果使用了公平锁，signalAll的与Condition关联的所有任务将以FIFO队列的形式获取锁，如果没有使用公平锁，则获取锁的任务是随机的，这样我们便可以更好地控制处在await状态的任务获取锁的顺序。与notifyAll相比，signalAll是更安全的方式。另外，它可以指定唤醒与自身Condition对象绑定在一起的任务。

下面将用条件变量实现生产者—消费者模型：

~~~java
// 定义信息类
class Info{	
    //定义name属性，为了与下面set的name属性区别开
	private String name = "name";
    // 定义content属性，为了与下面set的content属性区别开
	private String content = "content" ;
    // 设置标志位,初始时先生产
	private boolean flag = true ;	
	private Lock lock = new ReentrantLock();  
    //产生一个Condition对象
	private Condition condition = lock.newCondition(); 
	public  void set(String name,String content){
		lock.lock();
		try{
			while(!flag){
				condition.await() ;
			}
            // 设置名称
			this.setName(name) ;	
			Thread.sleep(300) ;
            // 设置内容
			this.setContent(content) ;	
            // 改变标志位，表示可以取走
			flag  = false ;	
			condition.signal();
		}catch(InterruptedException e){
			e.printStackTrace() ;
		}finally{
			lock.unlock();
		}
	}
 
	public void get(){
		lock.lock();
		try{
			while(flag){
				condition.await() ;
			}	
			Thread.sleep(300) ;
			System.out.println(this.getName() + 
				" --> " + this.getContent()) ;
            // 改变标志位，表示可以生产
			flag  = true ;	
			condition.signal();
		}catch(InterruptedException e){
			e.printStackTrace() ;
		}finally{
			lock.unlock();
		}
	}
 
	public void setName(String name){
		this.name = name ;
	}
	public void setContent(String content){
		this.content = content ;
	}
	public String getName(){
		return this.name ;
	}
	public String getContent(){
		return this.content ;
	}
}
// 通过Runnable实现多线程
class Producer implements Runnable{	
    // 保存Info引用
	private Info info = null ;		
	public Producer(Info info){
		this.info = info ;
	}
	public void run(){
        // 定义标记位
		boolean flag = true ;	
		for(int i=0;i<10;i++){
			if(flag){
                // 设置名称
				this.info.set("姓名--1","内容--1") ;	
				flag = false ;
			}else{
                // 设置名称
				this.info.set("姓名--2","内容--2") ;	
				flag = true ;
			}
		}
	}
}
class Consumer implements Runnable{
	private Info info = null ;
	public Consumer(Info info){
		this.info = info ;
	}
	public void run(){
		for(int i=0;i<10;i++){
			this.info.get() ;
		}
	}
}
public class ThreadCaseDemo{
	public static void main(String args[]){
        // 实例化Info对象
		Info info = new Info()
        // 生产者;	
		Producer pro = new Producer(info) ;	
        // 消费者
		Consumer con = new Consumer(info) ;	
		new Thread(pro).start() ;
		//启动了生产者线程后，再启动消费者线程
		try{
			Thread.sleep(500) ;
		}catch(InterruptedException e){
			e.printStackTrace() ;
		}
 
		new Thread(con).start() ;
	}
}

~~~

Lock和Condition对象，在处理更复杂的多线程问题时，会有明显的优势，只有在更加困难的多线程问题中才是必须的。

### Lock与synchronized

总结来说，Lock和synchronized有以下几点不同：

1. Lock是一个接口，而synchronized是Java中的关键字，synchronized是内置的语言实现；
2. synchronized在发生异常时，会自动释放线程占有的锁，因此不会导致死锁现象发生；而Lock在发生异常时，如果没有主动通过unLock()去释放锁，则很可能造成死锁现象，因此使用Lock时需要在finally块中释放锁；
3.  ReetrantLock有两种锁：忽略中断锁和响应中断锁。忽略中断锁与synchronized实现的互斥锁一样，不能响应中断，而响应中断锁可以响应中断。
4. 通过Lock可以知道有没有成功获取锁，而synchronized却无法办到。
5. Lock可以提高多个线程进行读操作的效率。

在性能上来说，如果竞争资源不激烈，两者的性能是差不多的，而当竞争资源非常激烈时（即有大量线程同时竞争），此时Lock的性能要远远优于synchronized。

### 参考资料

1. [java并发编程：Lock](https://www.cnblogs.com/dolphin0520/p/3923167.html)
2. [java并发编程：Lock锁和条件变量](https://blog.csdn.net/ns_code/article/details/17487337)





