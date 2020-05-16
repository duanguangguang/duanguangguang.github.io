---
title: JVM内存分配与回收策略
date: 2020-04-18 17:17:48
categories: jvm
tags:
  - jvm
---

### 介绍

介绍JVM内存分配和回收策略。

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\66JVM内存分配与回收策略.png)

<!-- more -->

### 一、内存分配

- 对象首先会进入Eden区

- 大对象直接进入老年代（示例：TestPolicy2）

  -XX:PretenureSizeThreshold=n，如果对象大于这个阈值，则直接进入老年代。

  注意：该参数只能在ParNew和Serial这两款垃圾收集器起作用

- -XX:SurvivoRatio=8

  新生代：Eden+Survivor0（S0）+Survivor1（S1） = Xmn

  -Xmn100m -XX:SurvivoRatio=8，请问Eden区多大？

  Eden：S0：S1 = SurvivoRatio：1：1

  答案：100*（8/（8+1+1）） = 80M

- 长期存活的对象进入老年代（示例：TestPolicy3）

  -XX:MaxTenuringThreshold，一个对象经历过多少次GC（MinorGC）会进入老年代，默认值15

- 对象年龄的动态判断

  示例同TestPolicy3，去掉VM参数-XX:TargetSurvivorRatio=90。

- 空间分配担保

  HandlePromotionFailure，检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将尝试着进行一次Minor GC；如果小于，或者设置不允许冒险，那这时也要改为进行一次Full GC。

  ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\空间分配担保.png)

  - 在Minor GC之前，检查老年代最大可用连续空间是否大于新生代所有对象的大小
  - 内存够，执行Minor GC
  - 如果空间不够，检查HandlePromotionFailure是否开启
  - 如果没有开启，这个时候执行Full GC
  - 如果开启了，检查老年代最大可用的连续空间是否大于历次晋升到老年代对象的平均大小
  - 如果大于，执行Minor GC
  - 如果小于，执行ull GC
  - 目的：减少Full GC

### 内存管理参数

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数1.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数2.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数3.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数4.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数5.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数6.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数7.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数8.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数9.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-ram\内存管理参数10.png)