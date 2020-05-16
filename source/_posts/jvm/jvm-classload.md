---
title: JVM类加载机制
date: 2020-04-18 17:15:31
categories: jvm
tags:
  - jvm
---

### 介绍

介绍JVM类加载机制。类加载机制生命周期：

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-classload\31类加载机制.png)

<!-- more -->

### 一、加载class文件到内存步骤

#### 第一步：加载

加载需要做三件事情：

1. 这个文件在哪个位置？是jar包还是class文件（格式）？

   ~~~java
   java TestClass
   java -jar
   ~~~

   获取二进制字节流。

2. 静态存储结构转化为方法区运行时数据结构
3. 在java堆中生成一个Class对象，相对于一个句柄，去访问方法区

#### 第二步：验证

1. 验证Class文件的标识：魔数Magic Number
2. 验证Class文件版本号
3. 验证常量池
   - 常量类型（14个）
   - 常量类型数据结构是否正确
   - UTF8是否符合标准
4. Class文件的每个部分（字段表、方法表等）是否正确
5. 元数据验证
   - final的验证
   - 父类验证
   - 继承验证
6. 字节码验证
   - 指令验证
7. 符号引用验证
   - 通过符号引用是否能够找到字段、方法、类

#### 第三步：准备

为类变量分配内存病区设置类变量的初始化阶段。只对static类变量进行内存分配。

- 类变量：一般称为静态变量
- 实例变量：当对象被实例化的时候，实例变量就跟着确定，随着对象销毁而销毁

~~~java
static int n = 2;
//初始化值是0，而不是2.因为这个时候还没执行任何java方法（<clinit>）

static final int n = 2;
//这个对应到常量池，ConstantValue，在准备阶段n必须被赋值为2
~~~

#### 第四步：解析

队符号引用进行解析。把符号引用指向直接引用。

- 直接引用：指向目标的指针，或者一个偏移量
- 符号引用：以字面量形式定义在常量池中

主要涉及：

- 类
- 接口
- 字段
- 方法（接口方法、类方法）

~~~java
CONSTANT_Class_info
CONSTANT_Fieldref_info
CONSTANT_Methodref_info
CONSTANT_InterfaceMethodref_info
CONSTANT_MethodType_info
CONSTANT_MethodHandler_info（方法虚表：vtable，接口虚表：itable）
CONSTANT_InvokeDynamic_info
~~~

匹配规则：简单名字+描述符，同时满足。

1. 字段解析

   ~~~java
   class A extends B implements C,D {
       private String str; //字段的解析
   }
   ~~~

   解析字段的顺序：`A--->C,D--->B--->Object`。

   - 在本类中去找有没有匹配的字段，找到则返回
   - 如果类有接口，往上层接口找匹配的字段，没有则下一步
   - 搜索父类查找有没有匹配的字段
   - 最后是到Object基类查找
   - 如果没找到：`java.lang.NoSuchFieldError`。
   - 如果找到了，但是没有权限（private）：`java.lang.IllegalAccessError`

2. 类方法解析

   ~~~java
   class A extends B implements C,D {
       private void inc(); //方法的解析
   }
   ~~~

   方法解析顺序：``A--->B--->C,D--->Object`。

   - 在本类中去找有没有匹配的方法
   - 父类中去找匹配的方法
   - 接口列表中去找匹配的方法
   - 如果没找到：`java.lang.NoSuchMethodError`
   - 如果找到了，但是没有权限（private）：`java.lang.IllegalAccessError`

3. 接口方法解析

   - 在本类中查找有没有匹配的方法，找到则返回
   - 父类接口递归查找
   - 如果没找到：`java.lang.NoSuchMethodError`
   - 接口都是public，不需要进行权限检查

#### 第五步：初始化

~~~java
//<init>：类的实例构造器，类的初始化
//<clinit>：静态变量，静态块的初始化，没有静态变量和静态块，则没有该初始化逻辑

class A {
    static int i = 2; //<clinit>
    static { //<clinit>
        System.out.println("");
    }
    int n; //<init>
}
~~~

#### 第六步：使用



#### 第七步：卸载



### 二、类加载器

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-classload\33类加载机制.png)

机制：双亲委派，目的：安全

1. 父类加载的不给子类加载
2. 一个类只能加载一次

### 三、双亲委派模式

![](C:\duanguangguang.github.io\source\_posts\jvm\jvm-classload\34双亲委派模式.png)

为什么需要双亲委任？

黑客自定义一个`java.util.List`类，该List类具有系统的List类一样的功能，只是在某个函数稍做修改。这个函数经常使用，假如在这个函数中植入一些“病毒代码”，并且通过自定义类加载器加入JVM中。

### 四、自定义类加载器

自定义类加载器打破双亲委任模式。

~~~java
public class CustomizeClassLoader {
    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        // 自定义类加载器
        ClassLoader loader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }

                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (IOException e) {
                    throw  new ClassNotFoundException();
                }
            }
        };

        String className = "com.example.dodd.jvm.CustomizeClassLoader";
        Object o1 = CustomizeClassLoader.class.getClassLoader().loadClass(className).newInstance();
        Object o2 = loader.loadClass(className).newInstance();

        System.out.println(o1 == o2);
        System.out.println(o1.equals(o2));
        // 判断两个对象相不相等，最重要的条件是不是一个类加载器
        System.out.println(o1 instanceof CustomizeClassLoader);
        System.out.println(o2 instanceof CustomizeClassLoader);

        System.out.println(o1.getClass().getClassLoader());
        System.out.println(o2.getClass().getClassLoader());
    }
}

// false
// false
// true
// false
// sun.misc.Launcher$AppClassLoader@18b4aac2
// com.example.dodd.jvm.CustomizeClassLoader$1@2f2c9b19

~~~





