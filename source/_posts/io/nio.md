---
title: nio
date: 2019-01-27 18:14:17
categories: 
 - IO
---

### 介绍

**Java NIO（New IO / Non Blocking IO）**，与原来的IO有同样的作用和目的。NIO支持面向缓冲区的、基于 

通道的IO操作。NIO将以更加高效的方式进行文件的读写操作。NIO 与 IO 的主要区别：

| IO                        | NIO                           |
| ------------------------- | ----------------------------- |
| 面向流（Stream Oriented） | 面向缓存区（Buffer Oriented） |
| 阻塞IO（Blocking IO）     | 非阻塞IO（Non Blocking IO）   |
| （无）                    | 选择器（Selectors）           |

<!-- more -->

### NIO 与 IO 的区别

![](nio\nio01.png)

### 通道(Channel)

由 java.nio.channels 包定义的。Channel 表示 IO 源与目标打开的连接。Channel 类似于传统的“流”。只不过 Channel 本身不能直接访问数据，Channel 只能与Buffer 进行交互。

若需要使用 NIO 系统，需要获取用于连接 IO 设备的通道以及用于容纳数据的缓冲区。然后操作缓冲区，对数据进行处理。

![](nio\nio03.png)

Java 为 Channel 接口提供的最主要实现类如下：

- FileChannel：用于读取、写入、映射和操作文件的通道
- DatagramChannel：通过 UDP 读写网络中的数据通道
- SocketChannel：通过 TCP 读写网络中的数据
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel

#### 获取通道

获取通道的一种方式是对支持通道的对象调用getChannel() 方法。支持通道的类如下：

- FileInputStream 
- FileOutputStream 
- RandomAccessFile 
- DatagramSocket 
- Socket 
- ServerSocket

获取通道的其他方式是使用 Files 类的静态方法 newByteChannel() 获取字节通道。或者通过通道静态方法open() 打开并返回指定通道。

#### 通道的数据传输

将 Buffer 中数据写入 Channel。例如：

~~~java
int bytesWritten = inChannel.write(buf);
~~~

从 Channel 读取数据到 Buffer。例如：

~~~java
int bytesRead = inChannel.read(buf);
~~~

### 缓冲区(Buffer)

一个用于特定基本数据类型的容器。由 java.nio 包定义的，所有缓冲区都是 Buffer 抽象类的子类。Java NIO 中的 Buffer 主要用于与 NIO 通道进行交互，数据是从通道读入缓冲区，从缓冲区写入通道中的。Buffer 就像一个数组可以保存多个相同类型的数据。根据数据类型不同(boolean 除外) ，有以下 Buffer 常用子类：

- ByteBuffer
- CharBuffer
- ShortBuffer
- IntBuffer
- LongBuffer
- FloatBuffer
- DoubleBuffer

都是通过如下方法获取一个 Buffer 对象：

~~~java
//创建一个容量为 capacity 的 XxxBuffer 对象
static XxxBuffer allocate(int capacity)
~~~

#### 缓冲区的基本属性

- 容量 (capacity) 

  表示 Buffer 最大数据容量，缓冲区容量不能为负，并且创建后不能更改

- 限制 (limit)

  第一个不应该读取或写入的数据的索引，即位于 limit 后的数据不可读写。缓冲区的限制不能为负，并且不能大于其容量

- 位置 (position)

  下一个要读取或写入的数据的索引。缓冲区的位置不能为负，并且不能大于其限制

- 标记 (mark)

  标记是一个索引，通过 Buffer 中的 mark() 方法指定 Buffer 中一个特定的 position

- 重置 (reset)

  可以通过调用 reset() 方法恢复到这个 position

- 标记、位置、限制、容量遵守以下不变式： 0 <= mark <= position <= limit <= capacity

缓存区的基本属性如下图：

![](nio\nio02.png)

#### 缓冲区的数据操作

Buffer 所有子类提供了两个用于数据操作的方法：get() 与 put() 方法

- 获取 Buffer 中的数据

  ~~~java
  // 读取单个字节
  get()
  // 批量读取多个字节到 dst 中
  get(byte[] dst)
  // 读取指定索引位置的字节(不会移动 position)
  get(int index)
  ~~~

- 放入数据到 Buffer 中

  ~~~java
  // 将给定单个字节写入缓冲区的当前位置
  put(byte b)
  // 将 src 中的字节写入缓冲区的当前位置
  put(byte[] src)
  // 将指定字节写入缓冲区的索引位置(不会移动 position)
  put(int index, byte b)
  ~~~

#### 直接与非直接缓冲区

字节缓冲区要么是直接的，要么是非直接的。如果为直接字节缓冲区，则 Java 虚拟机会尽最大努力直接在此缓冲区上执行本机 I/O 操作。在每次调用基础操作系统的一个本机 I/O 操作之前（或之后），虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。

- 直接缓冲区

  ![](nio\nio05.png)

- 非直接缓冲区

  ![](nio\nio04.png)

直接字节缓冲区可以通过调用此类的 allocateDirect() 工厂方法来创建。此方法返回的缓冲区进行分配和取消 

分配所需成本通常高于非直接缓冲区。直接缓冲区的内容可以驻留在常规的垃圾回收堆之外，因此，它们对 

应用程序的内存需求量造成的影响可能并不明显。所以，建议将直接缓冲区主要分配给那些易受基础系统的 

本机 I/O 操作影响的大型、持久的缓冲区。一般情况下，最好仅在直接缓冲区能在程序性能方面带来明显好 

处时分配它们。

直接字节缓冲区还可以通过 FileChannel 的 map() 方法 将文件区域直接映射到内存中来创建。该方法返回 

MappedByteBuffer 。Java 平台的实现有助于通过 JNI 从本机代码创建直接字节缓冲区。如果以上这些缓冲区 

中的某个缓冲区实例指的是不可访问的内存区域，则试图访问该区域不会更改该缓冲区的内容，并且将会在 

访问期间或稍后的某个时间导致抛出不确定的异常。

字节缓冲区是直接缓冲区还是非直接缓冲区可通过调用其 isDirect() 方法来确定。提供此方法是为了能够在 

性能关键型代码中执行显式缓冲区管理。

### 分散和聚集

- 分散读取（Scattering Reads）

  是指按照缓冲区的顺序，从 Channel 中读取的数据分散到多个 Buffer 中

- 聚集写入（Gathering Writes）

  是指按照缓冲区的顺序，写入 position 和 limit 之间的数据到 Channel 

### transferFrom()和transferTo()

- 将数据从源通道传输到其他 Channel 中：

  ~~~java
  RandomAccessFile fromFile = new RandomAccessFie("data/fromFile.txt","rw");
  // 获取FileChannel
  FileChannel fromChannel = fromFile.getChannel();
  
  RandomAccessFile toFile = new RandomAccessFie("data/toFile.txt","rw");
  FileChannel toChannel = toFile.getChannel();
  
  // 定义传输位置
  long position = 0L;
  
  // 最多传输的字节数
  long count = fromChannel.size();
  
  // 将数据从源通道传输到另一个通道中：
  toChannel.transferFrom(fromChannel, count, position);
  ~~~

- 将数据从源通道传输到其他 Channel 中：

  ~~~java
  RandomAccessFile fromFile = new RandomAccessFie("data/fromFile.txt","rw");
  // 获取FileChannel
  FileChannel fromChannel = fromFile.getChannel();
  
  RandomAccessFile toFile = new RandomAccessFie("data/toFile.txt","rw");
  FileChannel toChannel = toFile.getChannel();
  
  // 定义传输位置
  long position = 0L;
  
  // 最多传输的字节数
  long count = fromChannel.size();
  
  // 将数据从源通道传输到另一个通道中：
  fromChannel.transferTo(position, count, toChannel);
  ~~~


### NIO 的非阻塞式网络通信

传统的 IO 流都是阻塞式的。也就是说，当一个线程调用 read() 或 write() 时，该线程被阻塞，直到有一些数据被读取或写入，该线程在此期间不能执行其他任务。因此，在完成网络通信进行 IO 操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务器端需要处理大量客户端时，性能急剧下降。

#### 非阻塞模式

Java NIO 是非阻塞模式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。因此，NIO 可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。

#### 选择器（Selector）

选择器（Selector） 是 SelectableChannle 对象的多路复用器，Selector 可以同时监控多个 SelectableChannel 的 IO 状况，也就是说，利用 Selector 可使一个单独的线程管理多个 Channel。Selector 是非阻塞 IO 的核心。

SelectableChannle 的结构如下图：

![](nio\nio06.png)

- 选择器应用