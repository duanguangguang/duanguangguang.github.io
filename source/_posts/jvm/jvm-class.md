---
title: JVM-CLASS文件结构
date: 2020-04-18 17:14:59
categories: jvm
tags:
  - jvm
---

### 介绍

介绍Java类文件结构。Java到.class文件的过程：

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\java到class文件.png)

<!-- more -->

### 一、Class文件组成内容

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\class文件组成内容.png)

- 无符号数

  就是数值：1,2,3...

- 表

  ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\字段表结构.png)

  就是一个结构，表的结构类似：

  ~~~java
  xxx_XXX_info{
      u1
      u2
      u4
      ...
  }
  ~~~

#### 1. Class文件格式

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\class文件格式.png)

- u1：一个字节
- u2：两个字节
- u4：四个字节
- u8：八个字节

演示案例：

~~~java
public class TestM {
    private int m;

    public int inc() {
        return m + 1;
    }
}
~~~

对应的字节码文件：

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\TestM.png)

1. 魔数

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\魔数.png)

   第一个u4字节代表这是一个class文件：cafe babe（固定的）。

2. 版本

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\版本.png)

   第二个u4字节代表JDK编译的版本号。

   - JDK7：51
   - JDK8：52（34是16进制，转换成10进制是52）

3. 常量池

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\class文件结构之常量池.png)

   - 子面量

     ~~~java
     int i = 3;
     ~~~

     字面量就是=右边的东西（3）。

   - 符号引用

     包括：

     - 类和接口的全限定名
     - 字段的名称和描述符
     - 方法的名称和描述符

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\class文件结构常量池.png)

   常量池总共有21个常量（HEX：00 16）（22-1）

   常量池结构表-1

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\常量池表1.png)

   常量池结构表-2

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\常量池表2.png)

   分析：

   - 0a：===》10 （对应到常量池结构表查找tag值）代表的Methodref_info

     00 06，00 18 ===》#6，#24

   - 09：===》09 代表的Fieldref_info

     03，19 ===》#3，#25

   - 07：===》07 代表的Class_info

     1a ===》#26

   - ...

   验证：

   ~~~java
   javap -v TestM.class
       
   Constant pool:
      #1 = Methodref          #6.#24         // java/lang/Object."<init>":()V
      #2 = Fieldref           #3.#25         // com/example/dodd/jvm/TestM.m:I
      #3 = Class              #26            // com/example/dodd/jvm/TestM
      #4 = Methodref          #3.#24         // com/example/dodd/jvm/TestM."<init>":()V
      #5 = Methodref          #3.#27         // com/example/dodd/jvm/TestM.inc:()I
      #6 = Class              #28            // java/lang/Object
      #7 = Utf8               m
      #8 = Utf8               I
      #9 = Utf8               <init>
     #10 = Utf8               ()V
     #11 = Utf8               Code
     #12 = Utf8               LineNumberTable
     #13 = Utf8               LocalVariableTable
     #14 = Utf8               this
     #15 = Utf8               Lcom/example/dodd/jvm/TestM;
     #16 = Utf8               inc
     #17 = Utf8               ()I
     #18 = Utf8               main
     #19 = Utf8               ([Ljava/lang/String;)V
     #20 = Utf8               args
     #21 = Utf8               [Ljava/lang/String;
     #22 = Utf8               SourceFile
     #23 = Utf8               TestM.java
     #24 = NameAndType        #9:#10         // "<init>":()V
     #25 = NameAndType        #7:#8          // m:I
     #26 = Utf8               com/example/dodd/jvm/TestM
     #27 = NameAndType        #16:#17        // inc:()I
     #28 = Utf8               java/lang/Object
   
   ~~~

   - init和clinit

     init是实例化初始化方法。使用场景：

     - 调用new初始化对象的时候
     - 调用反射的时候newInstance()
     - 调用clone方法的时候
     - ObjectInpustream.getObject()序列化的时候

     clinit是类和接口的初始化。使用场景：

     ~~~java
     class Test{
         static int m = 3;
     }
     
     class Test{
         static int m;
         static{
             m = 3;
         }
     }
     ~~~

     所有的类变量初始化语句和静态初始化语句都被java编译收集到一起，采用clinit初始化方式。采用javap命令查看字节码：

     ~~~java
     // 类静态变量初始化后出现
     static {};
         descriptor: ()V
         flags: ACC_STATIC
         Code:
           stack=1, locals=0, args_size=0
              0: iconst_2
              1: putstatic     #3                  // Field a:I
              4: return
           LineNumberTable:
             line 10: 0
     ~~~

   - 解读示例：

     ~~~java
     public com.example.dodd.jvm.TestM();
         descriptor: ()V
         flags: ACC_PUBLIC
         Code:
           stack=1, locals=1, args_size=1
              0: aload_0
              1: invokespecial #1            // Method java/lang/Object."<init>":()V
              4: return
           LineNumberTable:
             line 8: 0
           LocalVariableTable:
             Start  Length  Slot  Name   Signature
                 0       5     0  this   Lcom/example/dodd/jvm/TestM;
     ~~~

     这是TestM的默认构造方法，首先通过invokespecial #1找到虚拟机的第一条指令#1。

     - 第一步，到常量池中找#1常量（#1 = Methodref          #6.#24）

       \#6 指向Class_info的索引项（u2类型数据）

       \#26 指向NameAndType的索引项（u2类型数据）

     - 第二步，找#6号常量（#6号常量是Class：java/lang/Object）

     - 第三步，找#24号常量（#9:#10）

       NameAndType的数据结构（u2：名称常量项的索引，u2：描述符常量项的索引）

     - 第四步，#9对应\<init>

     - 第五步，#10对应()V

     至此，搜索的结果就是：`V java/lang/Object.\<init>()`。

   ps：类名应该多长？不能超过256。

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\常量池结构表说明.png)

4. 访问标志

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\访问标志.png)

   - 字节码：`00 21`固定

     0x0001|0x0020=0x0021，这就是为什么字节码是`00 21`。

5. 类索引、父类索引、接口计数器、接口1、接口2...

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\类索引父类索引接口索引.png)

   字节码：`00 03 00 06 00 00  `。

   - `00 03`代表#3常量索引：`com/example/dodd/jvm/TestM`

   - `00 06`代表#6常量索引：`java/lang/Object`

   - `00 00`代表有几个接口，当前是0个

   - ...

     如果有接口，后续字节码表示接口的索引项，如果该类没有实现任何接口，则计数器值为0，后面的接口索引表不再占用任何字节。

6. 字段表集合

   TestClass.class

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\字段表集合.png)

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\字段表结构和访问标志.png)

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\全限定名简单名称以及描述符.png)

   关于方法表查找方法示例：

   - 1.找到访问控制access_flag（00 01--->public）
   - 2.找到简单名字name_index（00 17 ---> inc）
   - 3.找到描述符descriptor_index（00 18 --->()I）

   翻译过来：`public int inc()`。

   - 4.找到attribute_count（00 01--->代表有一个属性表）
   - 5.对照属性表attribute_info(u2,u4,u1*length)
   - 6.找到attribute_name_index（00 11--->Code）
   - 7.找到attribute_length（00 00 00 be--->190个u1）

   max_stacks：方法的栈深。

   max_locals：方法的本地变量数量。

   args_size：方法的参数有多少个（默认是this（无参也是1），如果方法是static那么就是0）。

   - 8.找到max_stacks最大栈深（00 01--->1 max_stacks）

   - 9.找到max_locals最大变量数（00 05---> 5 max_locals）

   - 10.找到code_length代码行数（00 00 00 18--->24）

   - 11.对应的字节码

     [参考JVM指令表]()

     - 0x04：iconst_1
     - 0x3c：istore_1
     - putstatic #3：b3 xx xx

   - 12.异常表

     ~~~java
     from to target
     0    4  8
     index:#2(java/lang/Exception)
     ~~~

     如果没有异常表：

     - debug断点的问题
     - 错误日志没有行号

7. 方法表集合

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\方法表集合.png)

8. 属性表集合

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\属性表结构和虚拟机规范预定义的属性.png)

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\虚拟机规范预定义的属性.png)

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\属性表集合code属性.png)

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\属性表集合异常表结构.png)

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\属性表集合之Exception.png)

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\虚拟机属性值LocalVariableTable.png)

   本地变量表（LocalVariableTable）：

   - start+length：一个本地变量的作用域
   - slot：几个槽来存储
   - name：简单名字（变量名对应 ）
   - signature：伪泛型，泛型擦除，对泛型的标识

   ![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-class\属性表之sourcefile.png)

