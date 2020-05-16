---
title: JDK工具
date: 2020-05-15 23:40:15
date: 2020-04-18 17:18:25
categories: jvm
tags:
  - jvm
---

### 介绍

介绍JDK工具。

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\76JDK工具.png)

<!-- more -->

### 一、JPS

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\77JPS.png)

- jps -?

### 二、 jstat

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\78jstat.png)

- jstat -?

- jstat -gc 1512 1000 50

  每一秒钟打印一次，打印50次。

jstat详细介绍：

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat01.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat02.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat03.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat04.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat05.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat06.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat07.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat08.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat09.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat10.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat11.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat12.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat13.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat14.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat15.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat16.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat17.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat18.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\jstat19.png)

### 三、jinfo

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\79实时查看调整参数jinfo.png)

### 四、jmap

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\80内存Dump工具jmap.png)

- 工具：Memory Analyzer Tool = MAT（分析hprof）

### 五、jstack

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-tools\81堆栈跟踪工具jstack.png)

- Attach Listener：负责接收外部命令
- Single Dispactcher：接收命令分发给不同模块
- Finalizer：执行用户finallizer方法的线程
- Reference Handler：处理对象引用

死锁分析：

~~~java
public class DeadLock extends Thread{
    private Object object;

    public void set0(Object object) {
        this.object = object;
    }

    public void run() {
        synchronized (this) {
            System.out.println("死锁begin");
            synchronized (object) {
                System.out.println("死锁object");
            }
            System.out.println("死锁end");
        }
    }

    public static void main(String[] args) {
        DeadLock dl = new DeadLock();
        TestThread thread = new TestThread(dl);
        dl.set0(thread);
        dl.start();
        thread.start();

    }

}

class TestThread extends Thread {
    private Object object;

    public TestThread(Object object) {
        this.object = object;
    }

    public void run() {
        synchronized (this) {
            System.out.println("thread begin ...");
            synchronized (object) {
                System.out.println("synchronized object");
            }
            System.out.println("thread end ...");
        }
    }
}
~~~

- jstack 14500

### 六、jconsole



### 七、jvisual





