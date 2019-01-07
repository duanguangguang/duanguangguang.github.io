---
title: volatile关键字
date: 2019-01-07 22:15:17
tags:
 - 并发包
 - 面试
categories: 
 - 并发包
---

### 介绍

使用锁的方式可以解决共享变量内存可见性问题，但是使用锁太笨重，因为它会带来线程的上下文切换开销。对于解决内存可见性问题，Java还提供了一种弱形式的同步，就是volatile关键字。

该关键字可以确保对一个变量的更新对其他 线程马上可见。当一个变量被声明为volatile时，线程在写入变量时不会把值缓存在寄存器中，而是会把值刷新回主内存。当其他线程读取该共享变量时，会从主内存重新获取最新值。

volatile保证内存可见性，但不保证操作的原子性。

<!-- more -->

### volatile内存语义

volatile内存语义和synchronized很相似，当线程写入volatile变量值时就等价于线程退出synchronized同步块（把写入工作内存的变量值同步到主内存），读取volatile变量值时就相当于进入到同步块（先清空本地内存变量值，再从主内存获取最新值）。

### volatile适用场景

1. 写入变量不依赖变量的当前值时。因为依赖的话就是，获取->计算->写入三部操作，不是原子性的，volatile不保证原子性。
2. 读写变量值时没有加锁。因为加锁本身已经保证了内存可见性。

### java指令重排序

java内存模型允许编译器和处理器对指令重排序提高运行性能，并且只会对不存在数据依赖性的指令重排序。例：

~~~java
int a = 1;
int b = 2;
int c = a + b;
~~~

变量c的值依赖a和b的值，所以重排序后能保证c的操作在a，b之后，但是a，b谁先执行就不一定，这在单线程下不存在问题。看一个多线程的例子：

~~~java
public class ReadWriteThread {
    private static int num = 0;
    private static boolean ready = false;
    
    public static class ReadThread extends Thread {
        @Override
        public void run() {
            while (!Thread.currentThread().isInterrupted()) {
                if (ready) { //(1)
                    System.out.println(num+num); //(2)
                }
                System.out.println("read thread ...");
            }
        }
    }

    public static class WriteThread extends Thread {
        @Override
        public void run() {
            num = 2; //(3)
            ready = true; //(4)
            System.out.println("writeThread set over ...");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ReadThread rt = new ReadThread();
        rt.start();
        
        WriteThread wt = new WriteThread();
        wt.start();
        
        Thread.sleep(10);
        rt.interrupt();
        System.out.println("main exit ...");
    }
}
~~~

由于代码（1）（2）（3）（4）之间不存在依赖关系，所以写线程的代码（3）（4）可能被重排序为先（4）后（3），那么执行（4）后，线程可能已经执行了（1）操作，并且在（3）执行前开始执行（2）操作，结果就是0。使用volatile修饰ready就可以避免指令重排序和内存可见性问题。

写volatile变量时，可以确保volatile写之前的操作不会被编译器重排序到volatile写之后。读volatile变量时，可以确保volatile读之后的操作不会被编译器重排序到volatile读之前。