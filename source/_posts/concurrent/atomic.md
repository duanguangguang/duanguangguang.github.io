---
title: JDK8新增原子操作类LongAdder
date: 2019-01-09 21:46:00
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

使用AtomicLong时，在高并发下大量线程会同时去竞争更新同一个原子变量，但是由于同时只有一个线程的CAS操作会成功，这就造成了大量线程竞争失败后通过无限循环不断进行自旋尝试CAS的操作，这会浪费CPU资源。

~~~java
public final long incrementAndGet() {
    for (;;) {
        long current = get();
        long next = current + 1;
        if (compareAndSet(current, next))
            return next;
    }
}
~~~

因此JDK8新增了一个原子性递增或者递减类LongAdder用来克服在高并发下使用AtomicLong的缺点。

<!-- more -->

### LongAdder原理

把一个变量分解为多个变量，让同样多的线程去竞争多个资源那么性能问题不就解决了，LongAdder就是这个思路。如图AtomicLong是多个线程同时竞争同一个变量：

![](atomic\aa01.png)

如图LongAdder则是内部维护多个变量，每个变量初始化都0，在同等并发量的情况下，争夺单个变量的线程量会减少这是变相的减少了争夺共享资源的并发量，另外多个线程在争夺同一个原子变量时候如果失败并不是自旋CAS重试，而是尝试获取其他原子变量的锁，最后获取当前值时候是把所有Cell变量的值累加后再加上base返回的。

![](atomic\aa02.png)

LongAdder维护了一个延迟初始化的原子性更新数组（默认情况下Cell数组是null）和一个基值变量base。由于Cells占用内存是相对比较大的，所以一开始并不创建，而是在需要时候在创建，也就是惰性加载。

当一开始判断Cell数组是null并且并发线程比较少时，所有的累加操作都是对base变量进行的。保持Cell数组的大小是2的N次方，在初始化时Cell数组中的Cell元素个数为2，数组的下标使用每个线程的hashcode值的掩码表示，数组里面的变量实体是Cell类型，Cell类型是AtomicLong的一个改进，用来减少缓存的争用，也就是解决伪共享问题。

对于大多数原子操作字节填充是浪费的，因为原子性操作都是无规律的分散在内存中进行的（多个原子性变量的内存地址是不连续的），多个原子变量被放入到同一个缓存行的可能性很小。但是原子性数组元素的内存地址是连续的，所以数组内的多个元素能经常共享缓存行，因此这里使用`@sun.misc.Contended`注解对Cell类进行字节填充，这防止了数组中多个元素共享同一个缓存行，在性能上是一个提升。

自旋锁cellsBusy用来初始化和扩容数组表使用，这里没有必要用阻塞锁，当一次线程发现当前下标的元素获取锁失败后，会尝试获取其他下表的元素的锁。

### LongAdder源码分析

1. LongAdder的结构是怎样的？

   ![](atomic\aa03.jpg)

   由该图可知，LongAdder类继承自Striped64类，在Striped64内部维护着三个变量。LongAdder的真实值其实是base的值与Cell数组里面所有Cell元素中的value值得累加，base是个基础值，默认为0。cellsBusy用来实现自旋锁，状态只有0和1，当创建Cell元素，扩容Cell数组或者初始化Cell数组时，使用CAS操作该变量来保证同时只有一个线程可以进行其中之一的操作。

2. 当前线程应该访问Cell数组里面的哪一个Cell元素？

   下面主要看下add方法的实现，从这个方法里面就可以找到其他问题的答案。

   ~~~java
   public void add(long x) {
       Cell[] as; long b, v; int m; Cell a;
       if ((as = cells) != null || !casBase(b = base, b + x)) { //（1）
           boolean uncontended = true;
           if (as == null || (m = as.length - 1) < 0 || //（2）
               (a = as[getProbe() & m]) == null || //（3）
               !(uncontended = a.cas(v = a.value, v + x))) //（4）
               longAccumulate(x, null, uncontended); //（5）
       }
   }
   ~~~

   代码（1）首先看cells是否为null，如果为null则当前在基础变量base上进行累加，这时候就类似AtomicLong的操作。

   如果cells不为null或者线程执行代码（1）的CAS操作失败了，则会去执行代码（2）。代码（2）和（3）决定当前线程应该访问cells数组里面的哪一个Cell元素，如果当前线程映射的元素存在则执行代码（4），使用CAS操作去更新分配的Cell元素的value值，如果当前线程映射的元素不存在但是CAS操作失败则执行代码（5）。代码（2）（3）（4）合起来就是获取当前线程应该访问的cells数组的Cell元素，然后进行CAS更新操作，只是在获取期间如果有些条件不满足则会跳转到代码（5）执行。

   另外，当前线程应该访问cells数组的哪一个Cell元素是通过getProbe()&m进行计算的，其中m是当前cells数组元素个数-1，getProbe()则用于获取当前线程中变量threadLocalRandomProbe的值，这个值一开始为0，在代码（5）里面会对其进行初始化。并且当前线程通过分配的Cell元素的cas函数来保证对Cell元素value值更新的原子性。这也是问题2和问题6的答案。

3. 如何初始化Cell数组？

   下面重点研究longAccumulate的代码逻辑，这是cells数组被初始化和扩容的地方。

   ~~~java
   final void longAccumulate(long x, LongBinaryOperator fn,
                                 boolean wasUncontended) {
       //(6)初始化当前线程的变量threadLocalRandomProbe的值
       int h;
       if ((h = getProbe()) == 0) {
           ThreadLocalRandom.current(); 
           h = getProbe();
           wasUncontended = true;
       }
       boolean collide = false;                
       for (;;) {
           Cell[] as; Cell a; int n; long v;
           if ((as = cells) != null && (n = as.length) > 0) { //（7）
               if ((a = as[(n - 1) & h]) == null) { //（8）
                   if (cellsBusy == 0) {       // Try to attach new Cell
                       Cell r = new Cell(x);   // Optimistically create
                       if (cellsBusy == 0 && casCellsBusy()) {
                           boolean created = false;
                           try {               // Recheck under lock
                               Cell[] rs; int m, j;
                               if ((rs = cells) != null &&
                                   (m = rs.length) > 0 &&
                                   rs[j = (m - 1) & h] == null) {
                                   rs[j] = r;
                                   created = true;
                               }
                           } finally {
                               cellsBusy = 0;
                           }
                           if (created)
                               break;
                           continue;           // Slot is now non-empty
                       }
                   }
                   collide = false;
               }
               else if (!wasUncontended)       // CAS already known to fail
                   wasUncontended = true;
               // 当前Cell存在，则执行CAS设置（9）
               else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                            fn.applyAsLong(v, x))))
                   break;
               // 当前Cell数组元素个数大于CPU个数（10）
               else if (n >= NCPU || cells != as)
                   collide = false;            // At max size or stale
               // 是否有冲突（11）
               else if (!collide)
                   collide = true;
               // 如果当前元素个数没有达到CPU个数并且有冲突则扩容（12）
               else if (cellsBusy == 0 && casCellsBusy()) {
                   try {
                       if (cells == as) {      // Expand table unless stale
                           //12.1
                           Cell[] rs = new Cell[n << 1];
                           for (int i = 0; i < n; ++i)
                               rs[i] = as[i];
                           cells = rs;
                       }
                   } finally {
                       //12.2
                       cellsBusy = 0;
                   }
                   //12.3
                   collide = false;
                   continue;                   // Retry with expanded table
               }
               //（13）为了能够找到一个空闲的Cell，重新计算hash值，xorshift算法生成随机数
               h = advanceProbe(h);
           }
           //（14）初始化Cell数组
           else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
               boolean init = false;
               try {                           // Initialize table
                   if (cells == as) {
                       //14.1
                       Cell[] rs = new Cell[2];
                       //14.2
                       rs[h & 1] = new Cell(x);
                       cells = rs;
                       init = true;
                   }
               } finally {
                   //14.3
                   cellsBusy = 0;
               }
               if (init)
                   break;
           }
           else if (casBase(v = base, ((fn == null) ? v + x :
                                       fn.applyAsLong(v, x))))
               break;                          // Fall back on using base
       }
   }
   ~~~

   当每个线程第一次执行到代码（6）时，会初始化当前线程变量threadLocalRandomProbe的值，这个变量在计算当前线程应该被分配到cells数组的哪一个Cell元素时会用到。

   cells数组初始化是在代码（14）中进行的，其中cellsBusy是一个标识，为0说明当前cells数组没有在初始化或者扩容，也没有在新建Cell元素，为1则说明cells数组在被初始化或者扩容或者在创建新的Cell元素，通过CAS操作来进行0或1状态的切换，这里使用casCellsBusy函数。假设当前线程通过CAS设置cellsBusy为1，则当前线程开始初始化操作，那么这时候其他线程就就不能进行扩容了。如代码（14.1）初始化cells数组元素个数为2，然后使用h&1计算当前线程应该访问cell数组的哪个位置，也就是使用当前线程的threadLocalRandomProbe变量值&（cells数组元素个数-1），然后标示cells数组已经被初始化，最后代码（14.3）重置了cellsBusy标记。显然这里没有使用CAS操作却是线程安全的，原因是cellsBusy是volatile类型的，这保证了变量的内存可见性，另外此时其他地方的代码没有机会修改cellsBusy的值。在这里初始化的cells数组里面的两个元素的值目前还是null。

4. Cell数组如何扩容？

   cells数组的扩容是在代码（12）中进行的，对cells扩容是有条件的，也就是代码（10）（11）的条件都不满足的时候。具体就是当前cells的元素个数小于当前机器CPU个数并且当前多个线程访问了cells中同一个元素，从而导致冲突使其中一个线程CAS失败时才会进行扩容操作。这里为何要涉及CPU个数？因为只有当每个CPU都运行一个线程时才会使多线程效果最佳，也就是当cells数组元素个数与CPU个数一致时，每个Cell都使用一个CPU进行处理，这时性能才是最佳的。代码（12）中的扩容操作先通过CAS设置cellsBusy为1，然后才能进行扩容。假如CAS成功则执行代码（12.1）将容量扩充为之前的2倍，并复制Cell元素到扩容后数组。另外，扩容后cells数组里面除了包含复制过来的元素外，还包含其他新元素，这些元素的值目前还是null。

   在代码（7）（8）中，当前线程调用add方法并根据当前线程的随机数threadLocalRandomProbe和cells元素个数计算要访问的Cell元素下标，然后如果发现对应下标元素的值为null，则新增一个Cell元素到cells数组，并且在将其添加到cells数组之前要竞争设置cellsBusy为1。

5. 线程访问分配的Cell元素有冲突后如何处理？

   代码（13）对CAS失败的线程重新计算当前线程的随机值threadLocalRandomProbe，以减少下次访问cells元素时的冲突机会。

6. 如何保证线程操作被分配的Cell元素的原子性？

   下面看一下Cell的构造：

   ~~~java
   @sun.misc.Contended static final class Cell {
       volatile long value;
       Cell(long x) { value = x; }
       final boolean cas(long cmp, long val) {
           return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
       }
   
       // Unsafe mechanics
       private static final sun.misc.Unsafe UNSAFE;
       private static final long valueOffset;
       static {
           try {
               UNSAFE = sun.misc.Unsafe.getUnsafe();
               Class<?> ak = Cell.class;
               valueOffset = UNSAFE.objectFieldOffset
                   (ak.getDeclaredField("value"));
           } catch (Exception e) {
               throw new Error(e);
           }
       }
   }
   ~~~

   其内部维护了一个被声明为volatile的变量，这里声明为volatile是因为线程操作value变量时没有使用锁，为了保证变量的内存可见性。cas函数通过CAS操作，保证了当前线程更新时被分配的Cell元素中value值得原子性。另外，Cell类使用@sun.misc.Contended修饰是为了避免伪共享。

7. 其他方法

   - long sum()

     ~~~java
     public long sum() {
         Cell[] as = cells; Cell a;
         long sum = base;
         if (as != null) {
             for (int i = 0; i < as.length; ++i) {
                 if ((a = as[i]) != null)
                     sum += a.value;
             }
         }
         return sum;
     }
     ~~~

     内部操作是累加所有Cell内部的value值后再累加base。由于计算总和时没有Cell数组进行加锁，所以在累加过程中可能有其他线程对Cell中的值进行了修改，也有可能对数组进行了扩容，所以sum返回值并不是非常精确的。

   - void reset()

     ~~~java
     public void reset() {
         Cell[] as = cells; Cell a;
         base = 0L;
         if (as != null) {
             for (int i = 0; i < as.length; ++i) {
                 if ((a = as[i]) != null)
                     a.value = 0L;
             }
         }
     }
     ~~~

     reset是重置操作，把base置为0，如果Cell数组有元素，则元素值置为0。

   - long sumThenReset()

     ~~~java
     public long sumThenReset() {
         Cell[] as = cells; Cell a;
         long sum = base;
         base = 0L;
         if (as != null) {
             for (int i = 0; i < as.length; ++i) {
                 if ((a = as[i]) != null) {
                     sum += a.value;
                     a.value = 0L;
                 }
             }
         }
         return sum;
     }
     ~~~

     使用sum累加对应的Cell值后，把当前Cell的值置为0，base置为0。这样，当多线程调用该方法时会有问题，比如第一个调用线程清空Cell的值，则后一个线程调用时累加的都是0。

   - long longValue()

     等价于sum()。

### LongAccumulator类原理探究

LongAdder是LongAccumulator的一个特例，调用LongAdder就相当于使用下面的方式调用LongAccumulator：

~~~java
LongAdder adder = new LongAdder();
LongAccumulator accumulator = new LongAccumulator(
    new LongBinaryOperator(){
        @Override
        public long applyAsLong(long left, long right){
            return left + right;
        }
    }
),0);
~~~

LongAccumulator的构造函数，其中accumulatorFunction是一个双目运算器接口，其根据输入的两个参数返回一个计算值，identity则是LongAccumulator累加器的初始值：

~~~java
public LongAccumulator(LongBinaryOperator accumulatorFunction,
                       long identity) {
    this.function = accumulatorFunction;
    base = this.identity = identity;
}
~~~

~~~java
@FunctionalInterface
public interface LongBinaryOperator {
    long applyAsLong(long left, long right);
}
~~~

LongAccumulator相比于LongAdder的不同之处在于，在调用casBase时LongAdder传递的是b+x，LongAccumulator则使用了r = function.applyAsLong(b = base,x)来计算：

~~~java
// LongAdder的add
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
// LongAccumulator的accumulate
public void accumulate(long x) {
    Cell[] as; long b, v, r; int m; Cell a;
    if ((as = cells) != null ||
        (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended =
              (r = function.applyAsLong(v = a.value, x)) == v ||
              a.cas(v, r)))
            longAccumulate(x, function, uncontended);
    }
}
~~~

另外，LongAdder在调用longAccumulate时传递的是null（fn），而LongAccumulator在调用longAccumulate时传递的是function（fn）。当fn为null时使用的v+x加法运算，这时候就等价于LongAdder，当fn不为null 时则使用传递的fn函数计算：

~~~java
 else if (casBase(v = base, ((fn == null) ? v + x :
                  fn.applyAsLong(v, x))))
~~~

总结：LongAccumulator提供了更强大的功能，可以让用户自定义累加规则。

### 参考文档

1. [LongAdder解析](https://www.jianshu.com/p/1a0d8ee3f5c8)

