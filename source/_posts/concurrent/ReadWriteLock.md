---
title: java并发包锁原理解析（二）：ReadWriteLock
date: 2018-12-30 15:56:24
tags:
 - 并发包
 - 锁
 - 面试
categories: 
 - 并发包
---

### 介绍

在上文中提到了Lock接口以及对象，使用它，很优雅的控制了竞争资源的安全访问，但是这种锁不区分读写，称这种锁为普通锁。为了提高性能，Java提供了读写锁，在读的地方使用读锁，在写的地方使用写锁，灵活控制，如果没有写锁的情况下，读是无阻塞的，在一定程度上提高了程序的执行效率。

读写锁允许访问共享数据的并发性高于互斥锁允许的并发性。它利用了这样一个事实：虽然一次只有一个线程可以修改共享数据，但在许多情况下，任何数量的线程都可以同时读取数据。理论上，使用读写锁所允许的并发性的增加将导致相互使用互斥锁的性能提高。实际上，这种并发性的增加只能在多处理器上完全实现，并且只有在共享数据的访问模式合适时才能实现。

读写锁是适用于写少读多的场景。

<!-- more -->

### ReadWriteLock接口

ReadWriteLock 描述的是：一个资源能够被多个读线程访问，或者被一个写线程访问，但是不能同时存在读写线程。也就是说读写锁使用的场合是一个共享资源被大量读取操作，而只有少量的写操作。

ReadWriteLock是一个接口，在它里面只定义了两个方法：

~~~java
public interface ReadWriteLock {
    Lock readLock();
 
    Lock writeLock();
}
~~~

一个用来获取读锁，一个用来获取写锁。也就是说将文件的读写操作分开，分成2个锁来分配给线程，从而使得多个线程可以同时进行读操作。**注意： 在同一线程中，持有读锁后，不能直接调用写锁的lock方法 ，否则会造成死锁。**

下面的ReentrantReadWriteLock实现了ReadWriteLock接口。

### ReentrantReadWriteLock

![](ReadWriteLock\rw01.png)

读写锁内部维护了一个readLock和一个writeLock，这两个类都是Lock的实现。

他们依赖Sync实现具体功能，而Sync继承自AQS，并且也提供了公平和非公平的实现。

ReentrantReadWriteLock 巧妙的使用 state 的高 16 位表示读状态，也就是获取改读锁的线程个数，低 16 位 表示获取到写锁的线程的可重入次数。并通过CAS对其进行操作实现了读写分离。

内部类Sync的一些关键属性和方法，源码如下：

~~~java
static final int SHARED_SHIFT   = 16;

//共享锁（读锁）状态单位值65536 
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
//共享锁线程最大个数65535
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;

//排它锁(写锁)掩码 二进制 15个1
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
//用来记录最后一个获取读锁的线程获取读锁的可重入次数
private transient HoldCounter cachedHoldCounter;
//用来记录第一个获取到读锁的线程
private transient Thread firstReader；
//用来记录第一个获取到读锁的线程获取读锁的可重入次数
private transient int firstReadHoldCount;
//用来存放除去第一个获取读锁线程外的其他线程获取读锁的可重入次数
private transient ThreadLocalHoldCounter readHolds = new ThreadLocalHoldCounter();

/** 返回读锁线程数  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** 返回写锁可重入个数 */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
~~~

类图中 firstReader用来记录第一个获取到读锁的线程，firstReadHoldCount则记录第一个获取到读锁的线程获取读锁的可重入数。cachedHoldCounter用来记录最后一个获取读锁的线程获取读锁的可重入次数。

Sync的内部类HoldCounter类的源码，如下：

~~~java
static final class HoldCounter {
    int count = 0;
    //线程id
    final long tid = getThreadId(Thread.currentThread());
}
~~~

readHolds 是ThreadLocal 变量，用来存放除去第一个获取读锁线程外的其他线程获取读锁的可重入次数，可知ThreadLocalHoldCounter继承了ThreadLocal，里面initialValue方法返回一个HoldCounter对象，源码如下：

~~~java
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
~~~

该类具有以下属性：

- 该类不会强制执行锁定访问的读取器或编写器首选项顺序；

- 公平模式

  公平锁 利用AQS的CLH队列，释放当前保持的锁（读锁或者写锁）时，优先为等待时间最长的那个写线程分配写入锁，当前前提是写线程的等待时间要比所有读线程的等待时间要长。同样一个线程持有写入锁或者有一个写线程已经在等待了，那么试图获取公平锁的（非重入）所有线程（包括读写线程）都将被阻塞，直到最先的写线程释放锁。如果读线程的等待时间比写线程的等待时间还有长，那么一旦上一个写线程释放锁，这一组读线程将获取锁。

- 非公平模式

  非公平锁（默认） 这个和独占锁的非公平性一样，由于读线程之间没有锁竞争，所以读操作没有公平性和非公平性，写操作时，由于写操作可能立即获取到锁，所以会推迟一个或多个读操作或者写操作。因此非公平锁的吞吐量要高于公平锁。

- 重入

  - 读写锁允许读线程和写线程按照请求锁的顺序重新获取读取锁或者写入锁。当然了只有写线程释放了锁，读线程才能获取重入锁。
  - 写线程获取写入锁后可以再次获取读取锁，但是读线程获取读取锁后却不能获取写入锁。
  - 另外读写锁最多支持65535个递归写入锁和65535个递归读取锁。

- 锁降级

  写线程获取写入锁后可以获取读取锁，然后释放写入锁，这样就从写入锁变成了读取锁，从而实现锁降级的特性。

- 锁升级

  读取锁是不能直接升级为写入锁的。因为获取一个写入锁需要释放所有读取锁，所以如果有两个读取锁视图获取写入锁而都不释放读取锁时就会发生死锁。

- 锁获取中断

  读取锁和写入锁都支持获取锁期间被中断。这个和独占锁一致。

- Condition支持

  写入锁提供了条件变量(Condition)的支持，这个和独占锁一致，但是读取锁却不允许获取条件变量，将得到一个UnsupportedOperationException异常。

- 重入数

  读取锁和写入锁的数量最大分别只能是65535，包括重入数，2^16-1=65536。

示例：如何在更新缓存后执行锁定降级

~~~java
class CachedData {
   Object data;
   volatile boolean cacheValid;
   final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

   void processCachedData() {
     rwl.readLock().lock();
     if (!cacheValid) {
        // Must release read lock before acquiring write lock
        rwl.readLock().unlock();
        rwl.writeLock().lock();
        try {
          // Recheck state because another thread might have
          // acquired write lock and changed state before we did.
          if (!cacheValid) {
            data = ...
            cacheValid = true;
          }
          // Downgrade by acquiring read lock before releasing write lock
          rwl.readLock().lock();
        } finally {
          rwl.writeLock().unlock(); // Unlock write, still hold read
        }
     }

     try {
       use(data);
     } finally {
       rwl.readLock().unlock();
     }
   }
 }
~~~

如果需要较为精确的控制缓存，使用ReentrantReadWriteLock倒也不失为一个方案。

### 写锁WriteLock 的获取和释放

1. void lock

   写锁是个独占锁，同时只有一个线程可以获取该锁。 如果当前没有线程获取到读锁和写锁则当前线程可以获取到写锁然后返回。 如果当前已经有线程取到读锁和写锁则当前线程则当前请求写锁线程会被阻塞挂起。

   另外写锁是可重入锁，如果当前线程已经获取了该锁，再次获取的只是简单的把可重入次数加1后直接返回。源码如下：

   ~~~java
   public void lock() {
       sync.acquire(1);
   }
   ~~~

   ~~~java
   public final void acquire(int arg) {
       // sync重写的tryAcquire方法
       if (!tryAcquire(arg) &&
           acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
           selfInterrupt();
   }
   ~~~

   如上代码，lock()内部调用了AQS的acquire方法，其中的tryAcquire是ReentrantReadWriteLock 内部 sync 类重写的，代码如下：

   ~~~java
   protected final boolean tryAcquire(int acquires) {
       Thread current = Thread.currentThread();
       int c = getState();
       int w = exclusiveCount(c);
       //（1） c!=0说明读锁或者写锁已经被某线程获取
       if (c != 0) {
           //（2）w=0说明已经有线程获取了读锁或者w!=0并且当前线程不是写锁拥有者，则返回false
               if (w == 0 || current != getExclusiveOwnerThread())
                   return false;
           //（3）说明某线程获取了写锁，判断可重入个数
               if (w + exclusiveCount(acquires) > MAX_COUNT)
                   throw new Error("Maximum lock count exceeded");
   
           //（4） 设置可重入数量(1)
               setState(c + acquires);
           return true;
       }
   
       //（5）第一个写线程获取写锁
           if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
               return false;
       setExclusiveOwnerThread(current);
       return true;
   }
   ~~~

   如上代码（1），如果当AQS状态值不为0 则说明当前已经有线程获取到了读锁或者写锁，代码（2）如果w == 0 说明状态值的低 16 为0，而状态值不为0，则说明高16位不为0，这暗示已经有线程获取了读锁，所以直接返回false。

   如果w != 0 说明当前已经有线程获取了该写锁，则看当前线程是不是该锁的持有者，如果不是则返回false。

   执行到代码（3）说明当前线程之前获取到了该锁，则判断该线程的可重入此时是不是超过了最大值，是则抛异常，否则执行代码（4）增加当前线程的可重入次数，然后返回true。

   如果AQS的状态值等于0，则说明目前没有线程获取到读锁和写锁，则实行代码（5）,

   其中对于ReentrantReadWriteLock的子类NofairSync的writerShouldBlock方法的非公平锁的实现源码如下：

   ~~~java
   final boolean writerShouldBlock() {
       return false; // writers can always barge
   }
   ~~~

   如果代码对于非公平锁来说固定返回false,则说明代码（5）抢占式执行CAS尝试获取写锁，获取成功则设置当前锁的持有者为当前线程返回true，否则返回false。

   对于ReentrantReadWriteLock的子类FairSync的writerShouldBlock方法的公平锁的实现源码如下：

   ~~~java
   final boolean writerShouldBlock() {
     return hasQueuedPredecessors();
   }
   ~~~

   使用 hasQueuedPredecessors 来判断当前线程节点是否有前驱节点，如果有则当前线程放弃获取写锁的权限直接返回 false。

2. void lockInterruptibly

   类似 lock() 方法，不同在于该方法对中断响应，也就是当其它线程调用了该线程的 interrupt() 方法中断了当前线程，当前线程会抛出异常 InterruptedException，源码如下：

   ~~~java
   public void lockInterruptibly() throws InterruptedException {
       sync.acquireInterruptibly(1);
   }
   ~~~

3. boolean tryLock

   尝试获取写锁，如果当前没有其它线程持有写锁或者读锁，则当前线程获取写锁会成功，然后返回 true。 如果当前已经其它线程持有写锁或者读锁则该方法直接返回 false，当前线程并不会被阻塞。

   如果当前线程已经持有了该写锁则简单增加 AQS 的状态值后直接返回 true。源码如下：

   ~~~java
   public boolean tryLock( ) {
       return sync.tryWriteLock();
   }
   ~~~

   ~~~java
   final boolean tryWriteLock() {
       Thread current = Thread.currentThread();
       int c = getState();
       if (c != 0) {
           int w = exclusiveCount(c);
           if (w == 0 || current != getExclusiveOwnerThread())
               return false;
           if (w == MAX_COUNT)
               throw new Error("Maximum lock count exceeded");
       }
       if (!compareAndSetState(c, c + 1))
           return false;
       setExclusiveOwnerThread(current);
       return true;
   }
   ~~~

   **写入锁tryLock()也就是tryWriteLock()成功的条件是: 没有写入锁或者写入锁是当前线程，并且尝试一次修改state成功。**这里使用的非公平策略

4. boolean tryLock(long timeout, TimeUnit unit) 

   如果尝试获取写锁失败则会把当前线程挂起指定时间，待超时时间到后当前线程被激活，如果还是没有获取到写锁则返回 false。

   另外该方法对中断响应, 也就是当其它线程调用了该线程的 interrupt() 方法中断了当前线程，当前线程会抛出异常 InterruptedException。源码如下：

   ~~~java
   public boolean tryLock(long timeout, TimeUnit unit)throws InterruptedException {
         return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
   ~~~

5. void unlock

   尝试释放锁，如果当前线程持有该锁，调用该方法会让该线程对该线程持有的 AQS 状态值减一，如果减去 1 后当前状态值为 0 则当前线程会释放对该锁的持有，否者仅仅减一而已。

   如果当前线程没有持有该锁调用了该方法则会抛出 IllegalMonitorStateException 异常 ，源码如下：

   ~~~java
   public void unlock() {
       sync.release(1);
   }
   ~~~

   ~~~java
   public final boolean release(int arg) {
       //调用ReentrantReadWriteLock中sync实现的tryRelease方法
       if (tryRelease(arg)) {
           //激活阻塞队列里面的一个线程
           Node h = head;
           if (h != null && h.waitStatus != 0)
               unparkSuccessor(h);
           return true;
       }
       return false;
   }
   protected final boolean tryRelease(int releases) {
       //（6） 看是否是写锁拥有者调用的unlock
       if (!isHeldExclusively())
           throw new IllegalMonitorStateException();
       //（7）获取可重入值，这里没有考虑高16位，因为写锁时候读锁状态值肯定为0
       int nextc = getState() - releases;
       boolean free = exclusiveCount(nextc) == 0;
       //（8）如果写锁可重入值为0则释放锁，否者只是简单更新状态值。
       if (free)
           setExclusiveOwnerThread(null);
       setState(nextc);
       return free;
   }
   ~~~

   如上代码 tryRelease 首先通过 isHeldExcusively判断是否当前线程是该写锁的持有者，如果不是则抛异常，否则执行代码（7）说明当前线程持有写锁，持有写锁说明状态值的高16位为0，所以这里nextc值就是当前线程写锁的剩余可重入次数。

   代码（8）判断当前可重入次数是否为0，如果free为true 说明可重入次数为0，则当前线程会释放对写锁的持有，当前锁的持有者设置为null。如果free 为false,则简单更新可重入次数。

### 读锁ReadLock的获取和释放

1. void lock

   获取读锁，如果当前没有其它线程持有写锁，则当前线程可以获取读锁，AQS 的高 16 位的值会增加 1，然后方法返回。否者如果其它有一个线程持有写锁，则当前线程会被阻塞。源码如下：

   ~~~java
   public void lock() {
      sync.acquireShared(1);
   }
   ~~~

   ~~~java
   public final void acquireShared(int arg) {
       //调用ReentrantReadWriteLock中的sync的tryAcquireShared方法
       if (tryAcquireShared(arg) < 0)
           //调用AQS的doAcquireShared方法
           doAcquireShared(arg);
   }
   ~~~

   如上代码读锁的lock方法调用了AQS的aquireShared方法，内部调用了 ReentrantReadWriteLock 中的 sync 重写的 tryAcquireShared 方法，源码如下：

   ~~~java
   protected final int tryAcquireShared(int unused) {
      //(1)获取当前状态值
       Thread current = Thread.currentThread();
       int c = getState();
   
       //(2)判断是否写锁被占用
       if (exclusiveCount(c) != 0 &&
           getExclusiveOwnerThread() != current)
           return -1;
   
       //（3）获取读锁计数
       int r = sharedCount(c);
       //（4）尝试获取锁，多个读线程只有一个会成功，不成功的进入下面fullTryAcquireShared进行重试
       if (!readerShouldBlock() &&
           r < MAX_COUNT &&
           compareAndSetState(c, c + SHARED_UNIT)) {
           //(5)第一个线程获取读锁
           if (r == 0) {
               firstReader = current;
               firstReaderHoldCount = 1;
           //(6)如果当前线程是第一个获取读锁的线程
           } else if (firstReader == current) {
               firstReaderHoldCount++;
           } else {
               //（7）记录最后一个获取读锁的线程或记录其它线程读锁的可重入数
               HoldCounter rh = cachedHoldCounter;
               if (rh == null || rh.tid != current.getId())
                   cachedHoldCounter = rh = readHolds.get();
               else if (rh.count == 0)
                   readHolds.set(rh);
               rh.count++;
           }
           return 1;
       }
       //(8)类似tryAcquireShared，但是是自旋获取
       return fullTryAcquireShared(current);
   }
   ~~~

   如上代码，首先获取了当前AQS的状态值，然后代码（2）看是否有其他线程获取到了写锁，如果是则直接返回了-1，然后调用AQS的doAcquireShared 方法把当前线程放入阻塞队列。

   否则执行到代码（3）得到获取到读锁的线程个数，到这里要说明目前没有线程获取到写锁，但是还是有可能有线程持有读锁，然后执行代码（4）

   1. 非公平锁的readerShouldBlock实现代码如下：

   ~~~java
   final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
   }
   ~~~

   ~~~java
   final boolean apparentlyFirstQueuedIsExclusive() {
      Node h, s;
      return (h = head) != null && (s = h.next)  != null && !s.isShared() && s.thread != null;
     }
   ~~~

   如上代码作用是如果队列里面存在一个元素，则判断第一个元素是不是正在尝试获取写锁，如果不是的话，则当前线程使用判断当前获取读锁线程是否达到了最大值，最后执行CAS操作设置AQS状态值的高 16 位值增加 1。代码（5）（6）记录第一个获取读锁的线程，并统计该线程获取读锁的可重入次数，代码（7）使用cachedHoldCounter 记录最后一个获取到读锁的线程，并同时该线程获取读锁的可重入次数，另外readHolds记录了当前线程获取读锁的可重入次数。

   如果readerShouldBlock 返回 true 则说明有线程正在获取写锁，则执行代码（8）fullTryAcquireShared 代码与 tryAcquireShared 类似，不同在于前者是通过循环自旋获取：

   ~~~java
   final int fullTryAcquireShared(Thread current) {
       HoldCounter rh = null;
       for (;;) {
           int c = getState();
           if (exclusiveCount(c) != 0) {
               if (getExclusiveOwnerThread() != current)
                   return -1;
               // else we hold the exclusive lock; blocking here
               // would cause deadlock.
           } else if (readerShouldBlock()) {
               // Make sure we're not acquiring read lock reentrantly
               if (firstReader == current) {
                   // assert firstReaderHoldCount > 0;
               } else {
                   if (rh == null) {
                       rh = cachedHoldCounter;
                       if (rh == null || rh.tid != getThreadId(current)) {
                           rh = readHolds.get();
                           if (rh.count == 0)
                               readHolds.remove();
                       }
                   }
                   if (rh.count == 0)
                       return -1;
               }
           }
           if (sharedCount(c) == MAX_COUNT)
               throw new Error("Maximum lock count exceeded");
           if (compareAndSetState(c, c + SHARED_UNIT)) {
               if (sharedCount(c) == 0) {
                   firstReader = current;
                   firstReaderHoldCount = 1;
               } else if (firstReader == current) {
                   firstReaderHoldCount++;
               } else {
                   if (rh == null)
                       rh = cachedHoldCounter;
                   if (rh == null || rh.tid != getThreadId(current))
                       rh = readHolds.get();
                   else if (rh.count == 0)
                       readHolds.set(rh);
                   rh.count++;
                   cachedHoldCounter = rh; // cache for release
               }
               return 1;
           }
       }
   }
   ~~~

2. void lockInterruptibly

   类似 lock() 方法，不同在于该方法对中断响应，也就是当其它线程调用了该线程的 interrupt() 方法中断了当前线程，当前线程会抛出异常 InterruptedException。

3. boolean tryLock

   尝试获取读锁，如果当前没有其它线程持有写锁，则当前线程获取写锁会成功，然后返回 true。如果当前已经其它线程持有写锁则该方法直接返回 false，当前线程并不会被阻塞。

   如果其它获取当前线程已经持有了该读锁则简单增加 AQS 的状态值高 16 位后直接返回 true。

   **读取锁tryLock()也就是tryReadLock()成功的条件是：没有写入锁或者写入锁是当前线程，并且读线程共享锁数量没有超过65535个。**

4. boolean tryLock(long timeout, TimeUnit unit)

   与 tryLock（）不同在于多了超时时间的参数，如果尝试获取读锁失败则会把当前线程挂起指定时间，待超时时间到后当前线程被激活，如果还是没有获取到读锁则返回 false。

   另外该方法对中断响应, 也就是当其它线程调用了该线程的 interrupt() 方法中断了当前线程，当前线程会抛出异常 InterruptedException。

5. void unlock

   释放锁。源码如下：

   ~~~java
   public void unlock() {
      sync.releaseShared(1);
   }
   ~~~

   ~~~java
   public final boolean releaseShared(int arg) {
           if (tryReleaseShared(arg)) {
               doReleaseShared();
               return true;
           }
           return false;
       }
   protected final boolean tryReleaseShared(int unused) {
       Thread current = Thread.currentThread();
       //如果当前线程是第一个获取读锁线程
       if (firstReader == current) {
           //如果可重入次数为1
           if (firstReaderHoldCount == 1)
               firstReader = null;
           else//否者可重入次数减去1
               firstReaderHoldCount--;
       } else {
           //如果当前线程不是最后一个获取读锁线程，则从threadlocal里面获取
           HoldCounter rh = cachedHoldCounter;
           if (rh == null || rh.tid != current.getId())
               rh = readHolds.get();
           //如果可重入次数<=1则清除threadlocal
           int count = rh.count;
           if (count <= 1) {
               readHolds.remove();
               if (count <= 0)
                   throw unmatchedUnlockException();
           }
           //可重入次数减去一
           --rh.count;
       }
   
       //循环直到自己的读计数-1 cas更新成功
       for (;;) {
           int c = getState();
           int nextc = c - SHARED_UNIT;
           if (compareAndSetState(c, nextc))
   
               return nextc == 0;
       }
   }
   ~~~

   总结ReentrantReadWriteLock：

   ![](ReadWriteLock\rw02.png)

### 应用示例

ReentrantReadWriteLocks可用于在某些类型的集合的某些用途中提高并发性。这通常是值得的，只有当预期集合很大时，由读取器线程比读写器线程访问更多，并且需要具有超过同步的开销的操作。例如，这是一个使用TreeMap的类，该类预计很大并且可以同时访问：

~~~java
class RWDictionary {
    private final Map<String, Data> m = new TreeMap<String, Data>();
    private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    private final Lock r = rwl.readLock();
    private final Lock w = rwl.writeLock();

    public Data get(String key) {
        r.lock();
        try { return m.get(key); }
        finally { r.unlock(); }
    }
    public String[] allKeys() {
        r.lock();
        try { return m.keySet().toArray(); }
        finally { r.unlock(); }
    }
    public Data put(String key, Data value) {
        w.lock();
        try { return m.put(key, value); }
        finally { w.unlock(); }
    }
    public void clear() {
        w.lock();
        try { m.clear(); }
        finally { w.unlock(); }
    }
 }
~~~

### 参考资料

1. [ReentrantReadWriteLock 源码分析](https://www.cnblogs.com/huangjuncong/p/9183761.html)
2. [Class ReentrantReadWriteLock](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html)
3. [Java中的读/写锁](http://ifeve.com/read-write-locks/)