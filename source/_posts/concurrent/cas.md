---
title: cas
date: 2018-12-22 18:25:21
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

**CAS**即Compare and Swap，是一种非阻塞算法，一个线程的失败或者挂起不应该影响其他线程的失败或挂起的算法。其JDK提供的非阻塞原子性操作，它通过硬件保证了比较-更新的原子性。java.util.concurrent包中借助CAS实现了区别于synchronouse同步锁的一种乐观锁。

使用synchronized关键字可以实现线程安全性，即内存可见性和原子性，但这样会导致线程上下文切换和增加重新调度的开销。

java提供了非阻塞的volatile关键字来解决共享变量的可见性问题，这弥补了锁带来的开销问题，但volatile只能保证共享变量的可见性，不能解决读-改-写等的原子性问题。

<!-- more -->

### 原子性操作

原子性操作是指执行一系列操作时，这些操作要么全部执行，要么全部不执行。如下方法，++i。

~~~java
public void inc() {
    ++i;
}
~~~

使用Javap -c命令查看汇编代码，如下。

~~~java
public void inc() {
    Code:
      0: aload_0
      1: dup
      2: getfield     
      5: lconst_1
      6: ladd
      7: putfield     
     10: return
}
~~~

由此可见，简单的++i由2，5，6，7四步组成：

- 2是获取当前i的值并放入栈顶
- 5是把常量1放入栈顶
- 6是把当前栈顶中两个值相加并把结果放入栈顶
- 7是把栈顶的结果赋给i变量

因此，java中简单的一句++i被转换成汇编后就不具有原子性了。这里保证多个操作的原子性可以使用synchronized来实现i的内存可见性，但更好的方式是使用非阻塞的CAS算法实现的原子性操作类。

### 非阻塞算法

非阻塞算法，即nonblocking algorithms，现代的CPU提供了特殊的指令，可以自动更新共享数据，而且能够检测到其他线程的干扰，而 compareAndSet() 就用这些代替了锁定。以原子类AtomicInteger为例，在没有锁的情况下是如何做到数据正确性的。

~~~java
private volatile int value;
~~~

在没有锁的机制下可能需要借助volatile原语，保证线程间的数据是可见的（共享的）。这样才获取变量的值的时候才能直接读取。

~~~java
public final int get() {
        return value;
}
~~~

然后来看看++i是怎么做到的。

**JDK7实现**：原子性设置value值为原始值+1，返回递增后的值

~~~java
public final int getAndIncrement() {
    while (true) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return next;
    }
}
~~~

JDK7中的循环逻辑已经被JDK8中的原子操作类UNsafe内置了，内置可以提高复用性。

**JDK8实现**：原子性设置value值为原始值+1，返回递增后的值

~~~java
public final int getAndIncrement() {
    return unsafe.getAndAddInt(this, valueOffset, 1);
}
~~~

通过调用Unsafe的getAndAddLong方法进行原子性操作，第一个参数是AtomicLong实例的引用，第二个参数是value变量在AtomicLong中的偏移量，第三个参数是要设置的第二个变量的值。JDK8中unsafe.getAndAddLong方法是：

~~~java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
~~~

内部调用unsafe.compareAndSwapInt方法，如果原子变量中的value值等于expect，则使用update值更新该值并返回true，都在返回false。

整体的过程就是这样子的，利用CPU的CAS指令，同时借助JNI来完成Java的非阻塞算法。其它原子操作都是利用类似的特性完成的。

其中，unsafe.compareAndSwapInt方法，类似：

~~~java
if (this == expect) {
    this = update
    return true;
} else {
	return false;
}
~~~

那么问题就来了，成功过程中需要2个步骤：比较this == expect，替换this = update，compareAndSwapInt如何这两个步骤的原子性呢？ 下面看下CAS的原理。

### CAS原理

JDK里面的Unsafe类提供了一系列的compareAndSwap*方法，以compareAndSwapInt方法为例：

`boolean compareAndSwapInt(Object obj, long valueOffset, int expect, int update)`

CAS有四个操作数，分别为：对象内存位置，对象中的变量的偏移量，变量预期值和新值。其操作含义是，如果对象obj中内存偏移量为valueOffset的变量值为expect，则使用新的值update替换旧值expect。此操作具有 volatile 读和写的内存语义。

下面分别从编译器和处理器的角度来分析,CAS 如何同时具有 volatile 读和 volatile 写的内存语义。

#### 编译器

编译器不会对 volatile 读与 volatile 读后面的任意内存操作重排序；编译器不会对 volatile 写与 volatile 写前面的任意内存操作重排序。组合这两个条件，意味着为了同时实现 volatile 读和 volatile 写的内存语义，编译器不能对 CAS 与 CAS 前面和后面的任意内存操作重排序。

#### 处理器

下面来分析在常见的 intel x86 处理器中，CAS 是如何同时具有 volatile 读和 volatile 写的内存语义的。

下面是 sun.misc.Unsafe 类的 compareAndSwapInt() 方法的源代码：

~~~java
public final native boolean compareAndSwapInt(Object o, long offset,
                                              int expected,
                                              int x);
~~~

可以看到这是个本地方法调用。这个本地方法在 openjdk 中依次调用的 c++ 代码为：unsafe.cpp，atomic.cpp 和 atomic*windows*x86.inline.hpp。这个本地方法的最终实现在 openjdk 的如下位置：openjdk-7-fcs-src-b147-27*jun*2011\openjdk\hotspot\src\os*cpu\windows*x86\vm\ atomic*windows*x86.inline.hpp（对应于 windows 操作系统，X86 处理器）。下面是对应于 intel x86 处理器的源代码的片段：

~~~java
// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}
~~~

如上面源代码所示，程序会根据当前处理器的类型来决定是否为 cmpxchg 指令添加 lock 前缀。如果程序是在多处理器上运行，就为 cmpxchg 指令加上 lock 前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略 lock 前缀（单处理器自身会维护单处理器内的顺序一致性，不需要 lock 前缀提供的内存屏障效果）。

intel 的手册对 lock 前缀的说明如下：

1. 确保对内存的读 - 改 - 写操作原子执行。在 Pentium 及 Pentium 之前的处理器中，带有 lock 前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。从 Pentium 4，Intel Xeon 及 P6 处理器开始，intel 在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（area of memory）在 lock 前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（cache line）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读 / 写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking），缓存锁定将大大降低 lock 前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。
2. 禁止该指令与之前和之后的读和写指令重排序。
3. 把写缓冲区中的所有数据刷新到内存中。

上面的第 2 点和第 3 点所具有的内存屏障效果，足以同时实现 volatile 读和 volatile 写的内存语义。

### CPU如何实现原子操作

处理器保证从系统内存当中读取或者写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。但是复杂的内存操作处理器不能自动保证其原子性，比如跨总线宽度，跨多个缓存行，跨页表的访问。但是处理器提供总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性。

1. 使用总线锁保证原子性

   第一个机制是通过总线锁保证原子性。如果多个处理器同时对共享变量进行读改写（i++就是经典的读改写操作）操作，那么共享变量就会被多个处理器同时进行操作，这样读改写操作就不是原子的，操作完之后共享变量的值会和期望的不一致。

   原因是有可能多个处理器同时从各自的缓存中读取变量i，分别进行加一操作，然后分别写入系统内存当中。那么想要保证读改写共享变量的操作是原子的，就必须保证CPU1读改写共享变量的时候，CPU2不能操作缓存了该共享变量内存地址的缓存。

   处理器使用总线锁就是来解决这个问题的。所谓总线锁就是使用处理器提供的一个LOCK＃信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住,那么该处理器可以独占使用共享内存。

2. 使用缓存锁保证原子性

   第二个机制是通过缓存锁定保证原子性。在同一时刻我们只需保证对某个内存地址的操作是原子性即可，但总线锁定把CPU和内存之间通信锁住了，这使得锁定期间，其他处理器不能操作其他内存地址的数据，所以总线锁定的开销比较大，最近的处理器在某些场合下使用缓存锁定代替总线锁定来进行优化。

   频繁使用的内存会缓存在处理器的L1，L2和L3高速缓存里，那么原子操作就可以直接在处理器内部缓存中进行，并不需要声明总线锁。所谓“缓存锁定”就是如果缓存在处理器缓存行中内存区域在LOCK操作期间被锁定，当它执行锁操作回写内存时，处理器不在总线上声言LOCK＃信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性，因为缓存一致性机制会阻止同时修改被两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时会起缓存行无效。

**但是有两种情况下处理器不会使用缓存锁定**。第一种情况是：当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行（cache line），则处理器会调用总线锁定。第二种情况是：有些处理器不支持缓存锁定。对于Inter486和奔腾处理器,就算锁定的内存区域在处理器的缓存行中也会调用总线锁定。

以上两个机制我们可以通过Inter处理器提供了很多LOCK前缀的指令来实现。比如位测试和修改指令BTS，BTR，BTC，交换指令XADD，CMPXCHG和其他一些操作数和逻辑指令，比如ADD（加），OR（或）等，被这些指令操作的内存区域就会加锁，导致其他处理器不能同时访问它。

### CAS在锁机制中的应用

锁机制保证了只有获得锁的线程能够操作锁定的内存区域。JVM内部实现了很多种锁机制，有偏向锁，轻量级锁和互斥锁，除了偏向锁，JVM实现锁的方式都用到的循环CAS，当一个线程想进入同步块的时候使用循环CAS的方式来获取锁，当它退出同步块的时候使用循环CAS释放锁。参考：Java SE1.6中的Synchronized](http://www.infoq.com/cn/articles/java-se-16-synchronized)。

### CAS缺点

1. ABA问题

   因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。ABA问题的解决思路就是使用版本号。在变量前面追加上版本号或时间戳，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。

   从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

   ~~~java
   public boolean compareAndSet(
                  V expectedReference,//预期引用
                  V newReference,//更新后的引用
                 int expectedStamp, //预期标志
                 int newStamp //更新后的标志
   
   )
   ~~~

2. 循环时间长开销大

   自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升，pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零。第二它可以避免在退出循环的时候因内存顺序冲突（memory order violation）而引起CPU流水线被清空（CPU pipeline flush），从而提高CPU的执行效率。

   注：

   - CPU流水线的工作方式就象工业生产上的装配流水线，在CPU中由5~6个不同功能的电路单元组成一条指令处理流水线，然后将一条X86指令分成5~6步后再由这些电路单元分别执行，这样就能实现在一个CPU时钟周期完成一条指令，因此提高CPU的运算速度。
   - 内存顺序冲突一般是由假共享引起，假共享是指多个CPU同时修改同一个缓存行的不同部分而引起其中一个CPU的操作无效，当出现这个内存顺序冲突时，CPU必须清空流水线。

3. 只能保证一个共享变量的原子操作

   当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了**AtomicReference****类来保证引用对象之间的原子性，你可以把多个变量放在一个对象里来进行CAS操作。

### concurrent 包的实现

由于 java 的 CAS 同时具有 volatile 读和 volatile 写的内存语义，因此 Java 线程之间的通信现在有了下面四种方式：

1. A 线程写 volatile 变量，随后 B 线程读这个 volatile 变量。
2. A 线程写 volatile 变量，随后 B 线程用 CAS 更新这个 volatile 变量。
3. A 线程用 CAS 更新一个 volatile 变量，随后 B 线程用 CAS 更新这个 volatile 变量。
4. A 线程用 CAS 更新一个 volatile 变量，随后 B 线程读这个 volatile 变量。

Java 的 CAS 会使用现代处理器上提供的高效机器级别原子指令，这些原子指令以原子方式对内存执行读 - 改 - 写操作，这是在多处理器中实现同步的关键。同时，volatile 变量的读 / 写和 CAS 可以实现线程之间的通信。把这些特性整合在一起，就形成了整个 concurrent 包得以实现的基石。仔细分析 concurrent 包的源代码实现，会发现一个通用化的实现模式：

1. 首先，声明共享变量为 volatile；
2. 然后，使用 CAS 的原子条件更新来实现线程之间的同步；
3. 同时，配合以 volatile 的读 / 写和 CAS 所具有的 volatile 读和写的内存语义来实现线程之间的通信。

AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic 包中的类），这些 concurrent 包中的基础类都是使用这种模式来实现的，而 concurrent 包中的高层类又是依赖于这些基础类来实现的。从整体来看，concurrent 包的实现示意图如下：

![](cas\cas01.png)

### 参考资料

1. [深入理解java内存模型](https://www.infoq.cn/article/java-memory-model-5)
2. [Java SE1.6中的Synchronized](http://www.infoq.com/cn/articles/java-se-16-synchronized)
3. [深入分析Volatile的实现原理](http://www.infoq.com/cn/articles/ftf-java-volatile)

