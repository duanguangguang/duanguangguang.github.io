---
title: JVM运行时数据区
date: 2020-04-18 17:15:57
categories: jvm
tags:
  - jvm
---

### 介绍

介绍JVM内存区域，即运行时数据区。JVM内存结构规范图如下：

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\37JVM内存结构规范.png)

- 静态编译

  把Java文件编译成字节码Class文件，这个时候Class文件以静态方式存在。

- 类加载器

  把Class字节码文件加载到内存中。

- 其他

  类的元数据：简单名字 + 描述符放在方法区中。

  new的对象放在堆中。

<!-- more -->

### 一、运行时数据区

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\38运行时数据区.png)

可以简单理解线程共享和私有，方法区和堆是存数据的地方，栈包括虚拟机栈，本地方法栈和程序计数器是执行逻辑的地方。

### 二、堆

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\39堆.png)

### 三、方法区

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\40方法区.png)

JIT：热点代码编译后存储存在方法区。

### 四、程序计数器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\41程序计数器.png)

程序计数器作用

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\程序计数器和指令寄存器.png)

### 五、栈

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\42JVM内存结构规范.png)

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\43栈.png)



### 六、JVM栈之局部变量表

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\45JVM栈之局部变量表.png)

### 七、JVM堆栈和方法区

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\46JVM堆栈和方法区.png)

### 八、JVM执行流程

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-data\47JVM执行流程.png)

