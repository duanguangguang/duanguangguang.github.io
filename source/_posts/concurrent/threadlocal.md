---
title: ThreadLocal
date: 2019-01-08 20:35:41
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

ThreadLocal是JDK包提供的，它提供了线程本地变量。创建一个ThreadLocal变量后，每个线程都会复制一个变量到自己的本地内存，当多个线程操作这个变量时，实际操作的是自己本地内存里面的变量，从而避免了线程安全问题。

<!-- more -->

### ThreadLocal的实现原理

![](threadlocal\tl01.png)

如上类图可见，Thread类中有一个threadLocals和inheritableThreadLocals 都是ThreadLocalMap类型的变量，而ThreadLocalMap是一个定制化的Hashmap,默认每个线程中这两个变量都为null,只有当线程第一次调用了ThreadLocal的set或者get方法的时候才会创建。

其实每个线程的本地变量不是存到ThreadLocal实例里面的，而是存放到调用线程的threadLocals变量里面。也就是说ThreadLocal类型的本地变量是存放到具体线程内存空间的。

ThreadLocal其实就是一个外壳，它通过set方法把value值放入调用线程threadLocals里面存放起来，当调用线程调用它的get方法的时候再从当前线程的threadLocals变量里面拿出来使用。如果调用线程如果一直不终止的话，那么这个本地变量会一直存放到调用线程的threadLocals变量里面，

因此，当不需要使用本地变量时候可以通过调用ThreadLocal变量的remove方法，从当前线程的threadLocals变量里面删除该本地变量。可能还有人会问threadLocals为什么设计为Map结构呢？很明显是因为每个线程里面可以关联多个ThreadLocal变量。

1. void set(T value)

   ~~~java
   public void set(T var1) {
      //（1）获取当前线程
      Thread var2 = Thread.currentThread();
      //(2) 当前线程作为key,去查找对应的线程变量，找到则设置
      ThreadLocal.ThreadLocalMap var3 = this.getMap(var2);
      if(var3 != null) {
          var3.set(this, var1);
      } else {
       //(3) 第一次调用则创建当前线程对应的Hashmap
       this.createMap(var2, var1);
      }
   }
   ~~~

   代码（1）首先获取调用线程，然后使用当前线程作为参数调用了 getMap(var2) 方法

   ~~~java
   ThreadLocal.ThreadLocalMap getMap(Thread var1) {
      return var1.threadLocals;
   }
   ~~~

   threadlocal变量是绑定到了线程的成员变量里面。

   如果getMap(var2) 返回不为空，则把 value 值设置进入到 threadLocals，也就是把当前变量值放入了当前线程的内存变量 threadLocals，threadLocals 是个 HashMap 结构，其中 key 就是当前 ThreadLocal 的实例对象引用，value 是通过 set 方法传递的值。

   如果 getMap(var2) 返回空那说明是第一次调用 set 方法，则创建当前线程的 threadLocals 变量

   ~~~java
   void createMap(Thread var1, T var2) {
      var1.threadLocals = new ThreadLocal.ThreadLocalMap(this, var2);
   }
   ~~~

2. T get()

   ~~~java
   public T get() {
      //(4)获取当前线程
      Thread var1 = Thread.currentThread();
      //（5）获取当前线程的threadLocals变量
      ThreadLocal.ThreadLocalMap var2 = this.getMap(var1);
      //（6）如果threadLocals不为null,则返回对应本地变量值
      if(var2 != null) {
         ThreadLocal.ThreadLocalMap.Entry var3 = var2.getEntry(this);
         if(var3 != null) {
            Object var4 = var3.value;
            return var4;
          }
       }
   　　 //（7）threadLocals为空则初始化当前线程的threadLocals成员变量。
       return this.setInitialValue();
   }
   ~~~

   代码（4）首先获取当前线程实例，如果当前线程的threadLocals变量不为null则直接返回当前线程的本地变量。否则执行代码（7）进行初始化

   ~~~java
   private T setInitialValue() {
      //(8)初始化为null
      Object var1 = this.initialValue();
      Thread var2 = Thread.currentThread();
      ThreadLocal.ThreadLocalMap var3 = this.getMap(var2);
      //(9)如果当前线程变量的threadLocals变量不为空
      if(var3 != null) {
         var3.set(this, var1);
      //（10）如果当前线程的threadLocals变量为空
      } else {
         this.createMap(var2, var1);
      }
      return var1;
   }
   ~~~

   如果当前线程的 threadLocals 变量不为空，则设置当前线程的本地变量值为 null，否者调用 createMap 创建当前线程的 createMap 变量。

3. void remove()

   ~~~java
   public void remove() {
      ThreadLocal.ThreadLocalMap var1 = this.getMap(Thread.currentThread());
      if(var1 != null) {
        var1.remove(this);
      }
   }
   ~~~

   如果当前线程的 threadLocals 变量不为空，则删除当前线程中指定 ThreadLocal 实例的本地变量。

总结：在每个线程内部都有一个threadLocals的成员变量，该变量的类型为HashMap，其中key为我们定义的ThreadLocal变量的this引用，value则为使用set方法设置的值。每个线程的本地变量存放在线程自己的内存变量threadlocals中，如果当前线程一直不消亡，那么这些本地变量会一直存在，所以可能会造成内存溢出。

### InheritableThreadLocal类

ThreadLocal 不支持继承的特性，为了解决该问题 InheritableThreadLocal 应运而生，InheritableThreadLocal 继承自 ThreadLocal，提供了一个特性，就是子线程可以访问到父线程中设置的本地变量。

~~~java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    public InheritableThreadLocal() {
    }
　　//（1）
    protected T childValue(T var1) {
        return var1;
    }
　　//（2）
    ThreadLocalMap getMap(Thread var1) {
        return var1.inheritableThreadLocals;
    }
　　//（3）
    void createMap(Thread var1, T var2) {
        var1.inheritableThreadLocals = new ThreadLocalMap(this, var2);
    }
}
~~~

代码（3）可知InheritableThreadLocal重写createMap方法，那么可以知道现在当第一次调用set方法时候创建的是当前线程的inhertableThreadLocals变量的实例，而不再是threadLocals。

代码（2）可以知道当调用get方法获取当前线程的内部map变量时候，获取的是inheritableThreadLocals，而不再是threadLocals。

关键地方来了，重写的代码（1）是何时被执行的，以及如何实现子线程可以访问父线程本地变量的。这个要从Thread创建的代码看起，Thread的默认构造函数以及Thread.java类的构造函数如下：

~~~java
public Thread(Runnable target) {
   init(null, target, "Thread-" + nextThreadNum(), 0);
}
private void init(ThreadGroup g, Runnable target, String name,
   long stackSize, AccessControlContext acc) {
   //...
   //(4)获取当前线程
   Thread parent = currentThread();
   //...
   //(5)如果父线程的inheritableThreadLocals变量不为null
   if (parent.inheritableThreadLocals != null)
   //(6)设置子线程中的inheritableThreadLocals变量
   this.inheritableThreadLocals                =ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
   this.stackSize = stackSize;
   tid = nextThreadID();
}
~~~

创建线程时候在构造函数里面会调用init方法，前面讲到了inheritableThreadLocal类get,set方法操作的是变量inheritableThreadLocals，所以这里inheritableThreadLocal变量就不为null,所以会执行代码(6)，下面看createInheritedMap方法源码，如下：

~~~java
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
   return new ThreadLocalMap(parentMap);
}
~~~

可以看到createInheritedMap内部使用父线程的inheritableThreadLocals变量作为构造函数创建了一个新的ThreadLocalMap变量，然后赋值给了子线程的inheritableThreadLocals变量，那么接着进入到ThreadLocalMap的构造函数里面做了什么，源码如下：

~~~java
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                //(7)调用重写的方法
                Object value = key.childValue(e.value);//返回e.value
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
~~~

如上代码所做的事情就是把父线程的inhertableThreadLocals成员变量的值复制到新的ThreadLocalMap对象，其中代码(7)调用了InheritableThreadLocal类重写的代码（1）。

总结：InheritableThreadLocal类通过重写代码(2)和(3)让本地变量保存到了具体线程的inheritableThreadLocals变量里面，线程通过InheritableThreadLocal类实例的set 或者 get方法设置变量时候就会创建当前线程的inheritableThreadLocals变量。当父线程创建子线程时候，构造函数里面就会把父线程inheritableThreadLocals变量里面的本地变量拷贝一份复制到子线程的inheritableThreadLocals变量里面。

子线程需要获取父线程的threadlocal变量的情况：

- 比如子线程需要使用存放在threadlocal变量中的用户登录信息；
- 比如一些中间件需要把统一的id追踪的整个调用链路记录下来。

### ThreadLocalMap

ThreadLocalMap是ThreadLocal的内部类，没有实现Map接口，用独立的方式实现了Map的功能，其内部的Entry也独立实现。

![](threadlocal\tl02.png)

在ThreadLocalMap中，也是用Entry来保存K-V结构数据的。但是Entry中key只能是ThreadLocal对象，这点被Entry的构造方法已经限定死了。

~~~java
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;
    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
~~~

Entry继承自WeakReference（弱引用，生命周期只能存活到下次GC前），但只有Key是弱引用类型的，Value并非弱引用。ThreadLocalMap的成员变量：

~~~java
static class ThreadLocalMap {
    /**
     * The initial capacity -- MUST be a power of two.
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * The table, resized as necessary.
     * table.length MUST always be a power of two.
     */
    private Entry[] table;

    /**
     * The number of entries in the table.
     */
    private int size = 0;

    /**
     * The next size value at which to resize.
     */
    private int threshold; // Default to 0
}
~~~

#### 解决Hash冲突

和HashMap的最大的不同在于，ThreadLocalMap结构非常简单，没有next引用，也就是说ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用线性探测的方式，所谓线性探测，就是根据初始key的hashcode值确定元素在table数组中的位置，如果发现这个位置上已经有其他key值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次判断，直至找到能够存放的位置。

ThreadLocalMap解决Hash冲突的方式就是简单的步长加1或减1，寻找下一个相邻的位置。

~~~java
/**
 * Increment i modulo len.
 */
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

/**
 * Decrement i modulo len.
 */
private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
~~~

显然ThreadLocalMap采用线性探测的方式解决Hash冲突的效率很低，如果有大量不同的ThreadLocal对象放入map中时发送冲突，或者发生二次冲突，则效率很低。

所以这里引出的良好建议是：每个线程只存一个变量，这样的话所有的线程存放到map中的Key都是相同的ThreadLocal，如果一个线程要保存多个变量，就需要创建多个ThreadLocal，多个ThreadLocal放入Map中时会极大的增加Hash冲突的可能。

#### set方法

~~~java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
~~~

后续分析

#### getEntry方法

~~~java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
~~~

一上来要是直接找到的话 那么直接返回这个entry，没有找到的话进行就开始向后探测这个entry 因为向后探测 还是有可能找到的。如果再向后探测过程中 如果·发现有 失效的Entry 将这个失效的位置清理掉 数组长度 减一，并且将从这个位置开始 向后寻找最后一个空位置，在这个区间将所有失效的entry都清除掉。

### ThreadLocalMap内存泄漏问题

由于ThreadLocalMap的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

其实在ThreadLocal的set、get和remove方法里面可以找到一些时机对这些key为null的entry进行清理，但这些清理不是必须发生的。ThreadLocalMap的remove的清理过程：

~~~java
private void remove(ThreadLocal<?> key) {
    //（1）计算当前ThreadLocal变量所在的table数组位置，尝试使用快速定位方法
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    //（2）这里使用循环是防止快速定位失败后，遍历table数组
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        //（3）找到
        if (e.get() == key) {
            //（4）找到则调用WeakReference的clear方法清除对ThreadLocal的弱引用
            e.clear();
            //（5）清除key为null的元素
            expungeStaleEntry(i);
            return;
        }
    }
}
~~~

~~~java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // (7)去掉对value的引用
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        //（7）如果key为null，则去掉对value的引用
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
~~~

代码（7）从当前元素的下标开始查看table数组里面是否有key为null的其他元素，有则清理。循环推出的条件是遇到table里面有null的元素。所以这里知道null元素后面的Entry里面key为null的元素不会被清理。

#### 如何避免泄漏

既然Key是弱引用，那么我们要做的事，就是在调用ThreadLocal的get()、set()方法时完成后再调用remove方法，将Entry节点和Map的引用关系移除，这样整个Entry对象在GC Roots分析后就变成不可达了，下次GC的时候就可以被回收。

~~~java
ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();
try {
    threadLocal.set(new Session(1, "dodd"));
    // 其它业务逻辑
} finally {
    threadLocal.remove();
}
~~~

### 应用场景

例：Hibernate的session获取场景：

~~~java
private static final ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();

//获取Session
public static Session getCurrentSession(){
    Session session =  threadLocal.get();
    //判断Session是否为空，如果为空，将创建一个session，并设置到本地线程变量中
    try {
        if(session ==null&&!session.isOpen()){
            if(sessionFactory==null){
                rbuildSessionFactory();// 创建Hibernate的SessionFactory
            }else{
                session = sessionFactory.openSession();
            }
        }
        threadLocal.set(session);
    } catch (Exception e) {
        // TODO: handle exception
    }

    return session;
}
~~~

每个线程访问数据库都应当是一个独立的Session会话，如果多个线程共享同一个Session会话，有可能其他线程关闭连接了，当前线程再执行提交时就会出现会话已关闭的异常，导致系统异常。此方式能避免线程争抢Session，提高并发下的安全性。

使用ThreadLocal的典型场景正如上面的数据库连接管理，线程会话管理等场景，只适用于独立变量副本的情况，如果变量为全局共享的，则不适用在高并发下使用。

### 总结

1. 每个ThreadLocal只能保存一个变量副本，如果想要上线一个线程能够保存多个副本以上，就需要创建多个ThreadLocal。
2. ThreadLocal内部的ThreadLocalMap键为弱引用，会有内存泄漏的风险。
3. 适用于无状态，副本变量独立后不影响业务逻辑的高并发场景。如果如果业务逻辑强依赖于副本变量，则不适合用ThreadLocal解决，需要另寻解决方案。

### 参考文档

1. [ThreadLocal-面试必问深度解析](https://www.jianshu.com/p/98b68c97df9b)
2. [理解java中的弱引用](https://droidyue.com/blog/2014/10/12/understanding-weakreference-in-java/)