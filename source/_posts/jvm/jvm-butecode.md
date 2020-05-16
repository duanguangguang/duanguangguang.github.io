---
title: JVM字节码执行引擎
date: 2020-04-18 17:18:25
categories: jvm
tags:
  - jvm
---

### 介绍

介绍JVM字节码执行引擎。

<!-- more -->

### 一、字节码执行引擎

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-butecode\68字节码执行引擎.png)

### 二、运行时栈帧结构

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-butecode\69运行时栈帧结构.png)

### 三、局部变量表

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-butecode\70局部变量表.png)

问题：long和double在32位操作系统是不是原子操作？

因为slot存放的是32位内的数据类型，在多线程情况下分高低位赋值和更新，不是原子性操作。

### 四、操作数栈

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-butecode\71操作数栈.png)

### 五、方法返回地址

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-butecode\72方法返回地址.png)

### 六、方法调用

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-butecode\73方法调用.png)

- 动态链接

- 静态链接

  符号引用 ---> 直接引用

- 非虚方法

  - invokestatic：静态方法
  - invokespecial：构造函数
  - invokevirtual：虚函数调用

- 虚方法

  需要我们在执行阶段确定需要执行的版本。

### 七、虚拟机动态分派机制

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-butecode\74虚拟机动态分派机制.png)

- 虚方法表

  一般是在类加载的阶段进行初始化。



