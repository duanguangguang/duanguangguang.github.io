---
title: JVM垃圾回收机制
date: 2020-04-18 17:16:27
categories: jvm
tags:
  - jvm
---

### 介绍

介绍JVM内存回收机制。对什么区域进行垃圾回收？

1. 堆
2. 方法区

栈是线程私有的，所有不进行回收。

查看GC回收情况：`-XX:+PrintGC`。

查看GC回收详情：`-XX:+PrintGCDetails`。

<!-- more -->

### 一、什么情况下回收？

1. 引用计数

   给对象添加一个引用计数器，每当对这个对象进行一次引用，计数器就加1；每当引用失效的时候，引用计数器就减1。当这个引用计数器等于0的时候，表示这个对象不会再被引用了。

2. 可达性分析

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\51可达性分析算法.png)

### 二、Java的引用类型

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\52java的引用类型.png)

1. 强引用

   在我们的代码中有明显的new Object()这类引用，只要这种引用还存在，垃圾回收器就不会回收它，就算内存不够。抛出OutOfMemoryError也不会回收这种对象。

2. 软引用

   如果内存够用的情况下，不会回收。如果内存不够用了才进行回收。

3. 弱引用

   生存到下一次垃圾回收之前，无论当前内存是否够用，都回收掉被弱引用关联的对象。

4. 虚引用

   不会对对象的生命周期有任何影响，也无法通过它得到对象的实例，唯一的作用也就是在对象被垃圾回收前收到一个系统通知。

### 三、标记清除算法

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\53标记清除算法.png)



### 四、复制算法

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\54复制算法.png)

### 五、标记整理算法

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\55标记整理算法.png)

### 六、分代算法

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\54分代算法.png)

### 七、JVM内存区域示意图

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\55JVM内存区域示意图.png)

### 八、JVM部分内存参数

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\49JVM部分内存参数.png)

### 九、安全点Safepoint

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\57安全点safepoint.png)

### 十、Serial收集器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\58Serial收集器.png)

单线程，新生代和老年代都会STW。

### 十一、ParNew收集器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\59ParNew收集器.png)

只对新生代并行收集，对老年代还是单线程收集，收集过程中STW。

### 十二、Parallel Scavenge收集器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\60Parallen-Scavenge收集器.png)

并行多线程收集器，他和ParNew的区别是用户可以控制GC时用户线程停顿的时间。

吞吐量=用户代码执行时间/（GC时间+用户代码执行时间）。

### 十三、Serial Old收集器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\61Serial-Old收集器.png)

### 十四、Parallel Old收集器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\62Parallel-Old收集器.png)

老年代收集器的多线程版本。

### 十五、CMS收集器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\63CMS收集器.png)

步骤：

- 初始标记（STW）

  扫描GCRoot对象。

- 并发标记

  标记GCRoot对象的子对象。

- 重新标记（STW）

  进行可达性分析。

- 并发清除

特点：

- 低停顿

参数：

- -XX:+UseCNSCompactAtFullCollection：GC执行完后做一次整理操作

- -XX:+CMSFullGCsBeforeCompaction：执行多少次FULLGC要做一次整理操作

- 默认：68%CMS默认空间（查资料）

  -XX:+CMSInitiatingOccupancyFraction：用来设置CMS空间参数。

  - MinorGC针对新生代
  - MajorGC FullGC针对老年代，MajorGC的时候会同时执行MinorGC（Parallel Scavenge除外）
  - FullGC=MajorGC+MinorGC

CMS回收线程默认数量：（CPU数量+3）/4。

执行的任务对CPU要求比较高，CPU核心数量又比较少。

### 十六、G1收集器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\64G1收集器.png)

也是属于分代收集器，只不过把内存划分为多个Region。

特点：

- 空间整合

算法：

- 既属于标记整理，也属于复制

YoungGC:

- 扫描根GC Roots
- 更新RememberSet记录回收对象的数据结构
- 检测RememberSet哪些数据是要从年轻代到老年代
- 拷贝对象，要么往幸存代，要么往老年代
- 清理工作

Mixed GC

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\MixedGC.png)

### 十七、其他

1. JDK1.7/1.8默认的垃圾回收器：Parallel Scavenge（新生代）+Parallel Old
2. JDK1.9默认的垃圾回收器：G1

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-gc\收集器组合方式.png)



