---
title: java并发包锁原理解析（四）：StampedLock
date: 2018-12-31 11:32:53
tags:
 - 并发包
 - 锁
 - 面试
categories: 
 - 并发包
---

### 介绍

**StampedLock**是java8在java.util.concurrent.locks新增的一个API 。

该类是一个读写锁的改进，它的思想是读写锁中读不仅不阻塞读，同时也不应该阻塞写：**在读的时候如果发生了写，则应当重读而不是在读的时候直接阻塞写！ **

因为在读线程非常多而写线程比较少的情况下，写线程可能发生饥饿现象，也就是因为大量的读线程存在并且读线程都阻塞写线程，因此写线程可能几乎很少被调度成功！当读执行的时候另一个线程执行了写，则读线程发现数据不一致则执行重读即可。

所以读写都存在的情况下，使用StampedLock就可以实现一种无障碍操作，即读写之间不会阻塞对方，但是写和写之间还是阻塞的！

<!-- more -->

###StampedLock

StampedLock锁提供了三种模式的读写控制：

#### 写锁writeLock 

是个排它锁或者叫独占锁，同时只有一个线程可以获取该锁，当一个线程获取该锁后，其它请求的线程必须等待，当目前没有线程持有读锁或者写锁的时候才可以获取到该锁，请求该锁成功后会返回一个stamp票据变量用来表示该锁的版本，当释放该锁时候需要unlockWrite并传递参数stamp。 

#### 悲观读锁readLock 

是个共享锁，在没有线程获取独占写锁的情况下，同时多个线程可以获取该锁，如果已经有线程持有写锁，其他线程请求获取该读锁会被阻塞。这里说的悲观是说在具体操作数据前悲观的认为其他线程可能要对自己操作的数据进行修改，所以需要先对数据加锁，这是在读少写多的情况下的一种考虑,请求该锁成功后会返回一个stamp票据变量用来表示该锁的版本，当释放该锁时候需要unlockRead并传递参数stamp。

#### 乐观读锁tryOptimisticRead 

是相对于悲观锁来说的，在操作数据前并没有通过CAS设置锁的状态，如果当前没有线程持有写锁，则简单的返回一个非0的stamp版本信息，获取该stamp后在具体操作数据前还需要调用validate验证下该stamp是否已经不可用，也就是看当调用tryOptimisticRead返回stamp后到到当前时间间是否有其他线程持有了写锁，如果是那么validate会返回0，否者就可以使用该stamp版本的锁对数据进行操作。由于tryOptimisticRead并没有使用CAS设置锁状态所以不需要显示的释放该锁。该锁的一个特点是适用于读多写少的场景，因为获取读锁只是使用与或操作进行检验，不涉及CAS操作，所以效率会高很多，但是同时由于没有使用真正的锁，在保证数据一致性上需要拷贝一份要操作的变量到方法栈，并且在操作数据时候可能其他写线程已经修改了数据，而我们操作的是方法栈里面的数据，也就是一个快照，所以最多返回的不是最新的数据，但是一致性还是得到保障的。 

####  **StampedLock的实现思想** 

StampedLockd的内部实现是基于CLH锁的，CLH锁原理：锁维护着一个等待线程队列，所有申请锁且失败的线程都记录在队列。一个节点代表一个线程，保存着一个标记位locked,用以判断当前线程是否已经释放锁。当一个线程试图获取锁时，从队列尾节点作为前序节点， 使用类似如下代码（一个空的死循环）判断前序节点是否已经成功的释放了锁：

~~~java
while(pred.locked){  }   
~~~

pred表示当前试图获取锁的线程的前序节点，如果前序节点没有释放锁，则当前线程就执行该空循环并不断判断前序节点的锁释放，即类似一个自旋锁的效果，避免被系统挂起。当循环一定次数后，前序节点还没有释放锁，则当前线程就被挂起而不再自旋，因为空的死循环执行太多次比挂起更消耗资源。

### 示列

下面是java doc提供的StampedLock一个例子 ：

~~~java
class Point {
   private double x, y;
   // 锁实例
   private final StampedLock sl = new StampedLock();
   // 排它锁-写锁（writeLock）
   void move(double deltaX, double deltaY) { 
     long stamp = sl.writeLock();
     try {
       x += deltaX;
       y += deltaY;
     } finally {
       sl.unlockWrite(stamp);
     }
   }
    
  //乐观读锁（tryOptimisticRead）
   double distanceFromOrigin() { 
     // 尝试获取乐观读锁（1）
     long stamp = sl.tryOptimisticRead(); 
     // 将全部变量拷贝到方法体栈内（2）将两个字段读入本地局部变量
     double currentX = x, currentY = y; 
      // 检查在（1）获取到读锁票据后，锁有没被其他写线程排它性抢占（3）
     if (!sl.validate(stamp)) {
        // 如果被抢占则获取一个共享读锁（悲观获取）（4）
        stamp = sl.readLock(); 
        try {
          currentX = x; // 将两个字段读入本地局部变量(5)
          currentY = y; // 将两个字段读入本地局部变量(5)
        } finally {
           // 释放共享读锁（6）
           sl.unlockRead(stamp);
        }
     }
      // 返回计算结果（7）
     return Math.sqrt(currentX * currentX + currentY * currentY);
   }
    
  // 使用悲观锁获取读锁，并尝试转换为写锁
   void moveIfAtOrigin(double newX, double newY) {
     // 这里可以使用乐观读锁替换（1）
     long stamp = sl.readLock();
     try {
       // 如果当前点在原点则移动（2）
       while (x == 0.0 && y == 0.0) {
         // 尝试将获取的读锁升级为写锁（3）
         long ws = sl.tryConvertToWriteLock(stamp); 
         // 升级成功，则更新票据，并设置坐标值，然后退出循环（4）
         if (ws != 0L) { //这是确认转为写锁是否成功
           stamp = ws; //如果成功 替换票据
           x = newX; //进行状态改变
           y = newY; //进行状态改变
           break;
         }
         else { 
           // 读锁升级写锁失败则释放读锁，显示获取独占写锁，然后循环重试（5）
           sl.unlockRead(stamp); //我们显式释放读锁
           stamp = sl.writeLock(); //显式直接进行写锁 然后再通过循环再试
         }
       }
     } finally {
       sl.unlock(stamp); //释放读锁或写锁(6)
     }
   }
 }
~~~

首先分析下move方法，该函数作用是在添加增量，改变当前point坐标的位置，代码先获取到了写锁，然后对point坐标进行修改，然后释放锁。该锁是排它锁，这保证了其他线程调用move函数时候会被阻塞，直到当前线程显示释放了该锁，也就是保证了对变量x,y操作的原子性。 

distanceFromOrigin方法，该方法作用是计算当前位置到原点的距离，代码（1）首先尝试获取乐观读锁，如果当前没有其它线程获取到了写锁，那么（1）会返回一个非0的stamp用来表示版本信息，代码（2）拷贝变量到本地方法栈里面，代码（3）检查在（1）获取到的票据是否还有效，之所以还要在此校验是因为代码（1）获取读锁时候并没有通过CAS操作修改锁的状态而是简单的通过与或操作返回了一个版本信息，这里校验是看在在获取版本信息到现在的时间段里面是否有其他线程持有了写锁，如果有则之前获取的版本信息就无效了。这里如果校验成功则执行（7）使用本地方法栈里面的值进行计算然后返回。需要注意的是在代码（3)校验成功后，代码（7）计算中其他线程可能获取到了写锁并且修改了x,y的值，而当前线程执行代码（7）进行计算时候采用的才是对修改前值的拷贝，也就是操作的值是对之前值的一个拷贝，并不是新的值。 

另外还有个问题，代码（2)和（3）能否互换，答案是不能，假设位置换了，那么首先执行validate，假如验证通过了，要拷贝x,y值到本地方法栈，而在拷贝的过程中很有可能其他线程已经修改了x,y中的一个，这就造成了数据的不一致性了。那么你可能会问，那不交换(2)和（3）时候在拷贝x,y值到本地方法栈里面时候也会存在其他线程修改了x,y中的一个值那，这个确实会存在，但是，别忘了拷贝后还有一道validate,如果这时候有线程修改了x,y中的值，那么肯定是有线程在调用validate前sl.tryOptimisticRead后获取了写锁，那么validate时候就会失败。现在应该明白了吧，这也是乐观读设计的精妙之处也是使用时候容易出问题的地方。下面继续分析validate失败后会执行代码（4）获取悲观读锁，如果这时候骑行线程持有写锁则代码（4）会导致的当前线程阻塞直到其它线程释放了写锁。获取到读锁后，代码（5）拷贝变量到本地方法栈，然后就是代码（6）释放了锁，拷贝的时候由于加了读锁在拷贝期间其它线程获取写锁时候会被阻塞，这保证了数据的一致性。最后代码（7）使用方法栈里面数据计算返回，同理这里在计算时候使用的数据也可能不是最新的，其它写线程可能已经修改过原来的x,y值了。 

最后一个方法moveIfAtOrigin方法作用是如果当前坐标为原点则移动到指定的位置。代码（1）获取悲观读锁，保证其它线程不能获取写锁修改x,y值，然后代码（2）判断如果当前点在原点则更新坐标，代码（3)尝试升级读锁为写锁，这里升级不一定成功，因为多个线程都可以同时获取悲观读锁，当多个线程都执行到（3）时候只有一个可以升级成功，升级成功则返回非0的stamp，否非返回0，这里假设当前线程升级成功，然后执行步骤（4）更新stamp值和坐标值然后退出循环，如果升级失败则执行步骤（5）首先释放读锁然后申请写锁，获取到写锁后在循环重新设置坐标值。最后步骤（6)释放锁。 

使用乐观读锁还是很容易犯错误的，必须要小心，必须要保证如下的使用顺序： 

~~~java
long stamp = lock.tryOptimisticRead(); //非阻塞获取版本信息
copyVaraibale2ThreadMemory();//拷贝变量到线程本地堆栈
if(!lock.validate(stamp)){ // 校验
    long stamp = lock.readLock();//获取读锁
    try {
        copyVaraibale2ThreadMemory();//拷贝变量到线程本地堆栈
     } finally {
       lock.unlock(stamp);//释放悲观锁
    }

}

useThreadMemoryVarables();//使用线程本地堆栈里面的数据进行操作
~~~

### StampedLock的特点

1. 所有获取锁的方法，都返回一个邮戳（Stamp），Stamp为0表示获取失败，其余都表示成功；
2. 所有释放锁的方法，都需要一个邮戳（Stamp），这个Stamp必须是和成功获取锁时得到的Stamp一致；
3. StampedLock是不可重入的；如果一个线程已经持有了写锁，再去获取写锁的话就会造成死；
4. StampedLock有三种访问模式：
   ①Reading（读模式）：功能和ReentrantReadWriteLock的读锁类似
   ②Writing（写模式）：功能和ReentrantReadWriteLock的写锁类似
   ③Optimistic reading（乐观读模式）：这是一种优化的读模式。
5. StampedLock支持读锁和写锁的相互转换
   我们知道RRW中，当线程获取到写锁后，可以降级为读锁，但是读锁是不能直接升级为写锁的。
   StampedLock提供了读锁和写锁相互转换的功能，使得该类支持更多的应用场景。
6. 无论写锁还是读锁，都不支持Conditon等待

### StampedLock与ReentrantReadWriteLock

ReentrantReadWriteLock 在沒有任何读写锁时，才可以取得写入锁，这可用于实现了悲观读取（Pessimistic Reading），即如果执行中进行读取时，经常可能有另一执行要写入的需求，为了保持同步，ReentrantReadWriteLock 的读取锁定就可派上用场。然而，如果读取执行情况很多，写入很少的情况下，使用 ReentrantReadWriteLock 可能会使写入线程遭遇饥饿（Starvation）问题，也就是写入线程吃吃无法竞争到锁定而一直处于等待状态。

StampedLock控制锁有三种模式（写，读，乐观读），一个StampedLock状态是由版本和模式两个部分组成，锁获取方法返回一个数字作为票据stamp，它用相应的锁状态表示并控制访问，数字0表示没有写锁被授权访问。在读锁上分为悲观锁和乐观锁。

所谓的乐观读模式，也就是若读的操作很多，写的操作很少的情况下，你可以乐观地认为，写入与读取同时发生几率很少，因此不悲观地使用完全的读取锁定，程序可以查看读取资料之后，是否遭到写入执行的变更，再采取后续的措施（重新读取变更信息，或者抛出异常） ，这一个小小改进，可大幅度提高程序的吞吐量！！

### 总结

1. synchronized是在JVM层面上实现的，不但可以通过一些监控工具监控synchronized的锁定，而且在代码执行时出现异常，JVM会自动释放锁定；
2. ReentrantLock、ReentrantReadWriteLock,、StampedLock都是对象层面的锁定，要保证锁定一定会被释放，就必须将unLock()放到finally{}中；
3. 相比ReentrantReadWriteLock读写锁，StampedLock是不可重入锁，通过提供乐观读锁在多线程多写线程少的情况下提供更好的性能，因为乐观读锁不需要进行CAS设置锁的状态而只是简单的测试状态 ；
4. StampedLock 对吞吐量有巨大的改进，特别是在读线程越来越多的场景下；
5. StampedLock有一个复杂的API，对于加锁操作，很容易误用其他方法；
6. 当只有少量竞争者的时候，synchronized是一个很好的通用的锁实现；
7. 当线程增长能够预估，ReentrantLock是一个很好的通用的锁实现。

### 参考资料

1. [Java 8新特性探究:StampedLock](http://www.importnew.com/14941.html)
2. [JDK8中StampedLock原理探究 ](http://ifeve.com/jdk8%E4%B8%ADstampedlock%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/)
3. [Java并发编程之StampedLock锁源码探究](https://juejin.im/entry/5b28a4c46fb9a00e8a3e534a)
4. [Java多线程进阶（十一）—— J.U.C之locks框架：StampedLock](https://segmentfault.com/a/1190000015808032)

