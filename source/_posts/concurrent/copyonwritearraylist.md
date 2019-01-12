---
title: 并发List源码解析：CopyOnWriteArrayList
date: 2019-01-12 13:35:55
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

并发包中的并发List只有CopyOnWriteArrayList，是一个线程安全的ArrayList，对其进行的修改操作都是在底层的一个复制的数组（快照）上进行的，也就是使用了写时复制策略。

<!-- more -->

### 类图结构

![](copyonwritearraylist\ca01.png)

在CopyOnWriteArrayList类图中，每个CopyOnWriteArrayList对象里面有一个array数组对象用来存放具体元素，ReentrantLock独占锁对象用来保证同时只有一个县城对array进行修改。

### 主要方法源码解析

#### 初始化

无参构造函数，在内部创建了一个大小为0的Object数组作为array的初始值。

~~~java
public CopyOnWriteArrayList() {
    setArray(new Object[0]);
}
~~~

有参构造函数，创建一个list，其内部元素是入参toCopyIn的副本。

~~~java
public CopyOnWriteArrayList(E[] toCopyIn) {
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
~~~

入参为集合，将集合里面的元素复制到副本list。

~~~java
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class)
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else {
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elements.getClass() != Object[].class)
            elements = Arrays.copyOf(elements, elements.length, Object[].class);
    }
    setArray(elements);
}
~~~

#### 添加元素

添加元素的方法有add(E e)、add(int index, E element)、addIfAbsent(E e)和addAllAbsent(Collection<? extends E> e)等，例：

~~~java
 public boolean add(E e) {
     //获取独占锁（1）
     final ReentrantLock lock = this.lock;
     lock.lock();
     try {
         //（2）获取array
         Object[] elements = getArray();
         //（3）复制array到新数组，添加元素到新数组
         int len = elements.length;
         Object[] newElements = Arrays.copyOf(elements, len + 1);
         newElements[len] = e;
         //（4）使用新数组替换添加前的数组
         setArray(newElements);
         return true;
     } finally {
         //（5）释放独占锁
         lock.unlock();
     }
 }
~~~

当一个线程通过（1）获取到锁后，就保证了在该线程添加元素的过程中其他线程不会对array进行修改；

代码（3）复制array到一个新数组，新数组的大小是原来数组大小增加1，所以CopyOnWriteArrayList是无界list，并把新增的元素添加到新数组。

代码（4）使用新数组替换原数组并在返回前释放锁。由于加锁，所以整个add过程是个原子操作。注意，添加元素，首先复制了一个快照，然后在快照上进行添加，而不是直接在原来数组上进行。

#### 获取指定位置元素

使用E get(int index)获取下标为index的元素，如果元素不存在则抛出IndexOutOfBoundsException异常。

~~~java
public E get(int index) {
    return get(getArray(), index);
}
~~~

~~~java
final Object[] getArray() {
    return array;
}
~~~

~~~java
private E get(Object[] a, int index) {
    return (E) a[index];
}
~~~

如上代码，当线程x调用get方法获取指定位置的元素时，分两步走，首先获取array数组（步骤A），然后通过下标访问指定位置的元素（步骤B），这两部操作过程并没有进行加锁同步，假如这时候List内容如下图所示，里面有1、2、3三个元素：

![](copyonwritearraylist\ca02.png)

由于执行步骤A和步骤B的过程没有加锁，这就可能导致在线程x执行完步骤A后执行步骤B前，另一个线程y进行了remove操作，假设要删除元素1。remove操作首先会获取独占锁。然后进行写事复制操作，也就是复制一份当前array数组，然后在复制的数组里面删除线程x通过get方法要访问的元素1，之后让array指向复制的数组。而这时候array之前指向的数组的引用计数为1而不是0，因为线程x还在使用它，这时线程x开始执行步骤B，步骤B操作的数组是线程y删除元素之前的数组，如下图所示：

![](copyonwritearraylist\ca03.png)

所以，虽然线程y已经删除了index处的元素，但是线程x的步骤B还是会返回index处的元素，这就是写时复制策略产生的弱一致性问题。

#### 修改指定元素

使用E set(int index, E element)修改list中指定元素的值，如果指定位置的元素不存在则抛出IndexOutOfBoundsException一场。

~~~java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
~~~

首先获取了独占锁，从而阻止其他线程对array数组进行修改，然后获取当前数组，并调用get方法获取指定位置的元素，如果指定位置的元素与新值不一致则创建新数组并复制元素，然后再新数组上修改指定位置的元素值并设置新数组到array。如果指定位置的元素值与新值一样，则为了保证volatile语义，还是要重新设置array，虽然array内容并没有改变。

#### 删除元素

删除list里面指定的元素，可以使用E remove(int index)、boolean remove(Object o)和boolean remove(Object o, Object[] snapshot, int index)等方法。例：

~~~java
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 获取数组
        Object[] elements = getArray();
        int len = elements.length;
        // 获取指定元素
        E oldValue = get(elements, index);
        int numMoved = len - index - 1;
        // 如果要删除的是最后一个元素
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1));
        else {
            // 分两次复制删除后剩余的元素到新数组
            Object[] newElements = new Object[len - 1];
            System.arraycopy(elements, 0, newElements, 0, index);
            System.arraycopy(elements, index + 1, newElements, index,
                             numMoved);
            // 使用新数组代替老数组
            setArray(newElements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
~~~

首先获取独占锁，然后获取数组中要被删除的元素，并把剩余的元素复制到新数组，之后使用新数组代替原来的数组，最后在返回前释放锁。

### 弱一致性的迭代器

CopyOnWriteArrayList中迭代器的弱一致性，所谓弱一致性是指返回迭代器后，其他线程对list的增删改对迭代器是不可见的。下面看看这是如何做到的：

~~~java
public Iterator<E> iterator() {
    return new COWIterator<E>(getArray(), 0);
}
~~~

~~~java
static final class COWIterator<E> implements ListIterator<E> {
    // array的快照版本
    private final Object[] snapshot;
    // 数组下标
    private int cursor;

    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }

    //是否遍历结束
    public boolean hasNext() {
        return cursor < snapshot.length;
    }

    public boolean hasPrevious() {
        return cursor > 0;
    }

    // 获取元素
    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext())
            throw new NoSuchElementException();
        return (E) snapshot[cursor++];
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }

    public int nextIndex() {
        return cursor;
    }

    public int previousIndex() {
        return cursor-1;
    }
}
~~~

在如上代码中，当调用iterator()方法获取迭代器时实际上会返回一个CowIterator对象，CowIterator对象的snapshot变量保存了当前list的内容，cursor是遍历list时数组的下标。

为什么说snapshot是list的快照呢？明明是指针传递的引用啊，而不是副本。如果在该线程使用返回的迭代器遍历元素的过程中，其他线程没有对list进行增删改，那么snapshot本身就是list的array，因为他们是引用关系。但是如果在遍历期间其他线程对该list进行了增删改，那么snapshot就是快照，因为增删改后list里面的数组被新数组替换了，这时候老数组被snapshot引用。这也说明获取迭代器后，使用迭代器元素时，其他线程对该list进行的增删改不可见，因为它们操作的是两个不同的数组，这就是弱一致性。

下面看一下多线程下迭代器的偌弱一致性的效果，例：

~~~java
public class TestCopyOnWriteArrayList {
    private static volatile CopyOnWriteArrayList<String> arrayList = new CopyOnWriteArrayList<>();

    public static void main(String[] args) throws InterruptedException {
        arrayList.add("hello");
        arrayList.add("world");
        arrayList.add("welcome");
        arrayList.add("to");
        arrayList.add("beijing");

        // 修改list中下标为1的元素为word
        Thread threadOne = new Thread(()->{
           arrayList.set(1,"word");
           // 删除元素
            arrayList.remove(2);
            arrayList.remove(3);
        });

        // 保证在修改线程启动前获取迭代器
        Iterator<String> itr = arrayList.iterator();

        // 启动线程
        threadOne.start();

        // 等待子线程执行完毕
        threadOne.join();

        //迭代元素
        while (itr.hasNext()) {
            System.out.println(itr.next());
        }

    }
}
~~~

输出结果：

~~~java
hello
world
welcome
to
beijing
~~~

主线程在子线程执行完毕后使用获取的迭代器遍历数组元素，从输出结果知道，在子线程里面进行的操作都没有生效，这就是弱一致性的体现。需要注意的是，获取迭代器的操作必须在子线程操作之前进行。

###  CopyOnWrite的缺点

1. 内存占用问题

   因为CopyOnWrite的写时复制机制，所以在进行写操作的时候，内存里会同时驻扎两个对象的内存，旧的对象和新写入的对象（注意:在复制的时候只是复制容器里的引用，只是在写的时候会创建新对象添加到新容器里，而旧容器的对象还在使用，所以有两份对象内存）。如果这些对象占用的内存比较大，比如说200M左右，那么再写入100M数据进去，内存就会占用300M，那么这个时候很有可能造成频繁的Yong GC和Full GC。之前我们系统中使用了一个服务由于每晚使用CopyOnWrite机制更新大对象，造成了每晚15秒的Full GC，应用响应时间也随之变长。

2. 数据一致性问题

   CopyOnWrite容器只能保证数据的最终一致性，不能保证数据的实时一致性。所以希望写入的的数据，马上能读到，则不能使用CopyOnWrite容器。

### 总结

CopyOnWriteArrayList使用写时复制的策略来保证list的一致性，而获取-修改-写入三步操作并不是原子性的，所以在增删改的过程中都使用了独占锁，来保证在某一时刻只有一个线程能对list数组进行修改。另外CopyOnWriteArrayList提供了弱一致性的迭代器，从而保证在获取迭代器后，其他线程对list的修改是不可见的，迭代器遍历的数组是一个快照。另外，CopyOnWriteArraySet的底层就是使用它实现的。

### 参考文档

1. [CopyOnWriteArrayList和同步容器的性能](http://blog.csdn.net/wind5shy/article/details/5396887)
2. [CopyOnWriteArrayList的使用](http://blog.csdn.net/imzoer/article/details/9751591)



