---
title: JVM调优实战
date: 2020-04-18 17:18:46
categories: jvm
tags:
  - jvm
---

### 介绍

介绍JVM调优。

非堆内存：

- 堆外内存
- 文件句柄
- Socket句柄
- 数据库连接

<!-- more -->

### 一、JVM调优原则

- 降低Full GC的频率，一天有1-2次，控制在晚上，重启服务器或者定时任务出发Full GC

- 确保大多数对象朝生夕死

- 提高大对象进入老年代的门槛（-XX:MaxTenuringThreshold=15）

- -Xms500m -Xmx500m设置成一样

  最小堆内存，最大堆内存，减少扩容过程耗时

- CMSInitiatingOccupancyFraction=（老年代-Eden-1个Survivor）/（老年的-新生代）*100

  设置CMS收集器在老年代空间被使用多少后出发垃圾收集。

- CMS调优示例

  ~~~java
  -server -Xmn500m -Xms1200m -Xmx1200m -XX:PermSize=256m -XX:MaxPermSize=256m
  -XX:SurvivorRation=1 -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70 
  -XX:+UseCMSCompactAtFullCollection -XX:+CMSFullGCsBeforeCompaction=5
  -XX:+HeapDumpOnOutOfMemoryError -Xloggc:log/gc.log -XX:+PrintGCDetails
  -XX:+CMSParallelRamarkEnabled
  ~~~

  - jstat -gc 2924
  - jstat -gccause 2924 1000
  - jstat -class 2924