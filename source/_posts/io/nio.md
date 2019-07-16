---
title: nio
date: 2019-07-14 15:14:17
categories: 
 - IO
---

### 介绍

**Java NIO（New IO / Non Blocking IO）**，是从Java 1.4版本开始引入的一个新的IO API，可以替代标准的Java IO API。NIO与原来的IO有同样的作用和目的。NIO支持面向缓冲区的、基于通道的IO操作。NIO将以更加高效的方式进行文件的读写操作。NIO 与 IO 的主要区别：

| IO                        | NIO                           |
| ------------------------- | ----------------------------- |
| 面向流（Stream Oriented） | 面向缓存区（Buffer Oriented） |
| 阻塞IO（Blocking IO）     | 非阻塞IO（Non Blocking IO）   |
| （无）                    | 选择器（Selectors）           |

<!-- more -->

### NIO 与 IO 的区别

![](nio\nio01.png)

### 通道(Channel)

由 java.nio.channels 包定义的。Channel 表示 IO 源与目标打开的连接。用于源节点与目标节点的连接。在 Java NIO 中负责缓冲区中数据的传输。Channel 本身不存储数据，因此需要配合缓冲区进行传输。

![](nio\nio03.png)

Java 为 Channel 接口提供的最主要实现类如下：

- FileChannel：用于读取、写入、映射和操作文件的通道 `（用于本地）`
- DatagramChannel：通过 UDP 读写网络中的数据通道 `（用于网络 UDP）`
- SocketChannel：通过 TCP 读写网络中的数据 `（用于网络 TCP）`
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel `（用于网络 TCP）`

#### 获取通道

获取通道的一种方式是对支持通道的对象调用getChannel() 方法。支持通道的类如下：

- 本地 IO：
  - FileInputStream 
  - FileOutputStream 
  - RandomAccessFile  `（随机存储文件流）`
- 网络IO：
  - DatagramSocket 
  - Socket 
  - ServerSocket

获取通道的其他方式：

~~~java
// 在 JDK 1.7 中的 NIO.2 针对各个通道提供了静态方法 open()
// 在 JDK 1.7 中的 NIO.2 的 Files 工具类的 newByteChannel()
~~~

#### 通道的数据传输

将 Buffer 中数据写入 Channel。例如：

~~~java
int bytesWritten = inChannel.write(buf)；
~~~

从 Channel 读取数据到 Buffer。例如：

~~~java
int bytesRead = inChannel.read(buf);
~~~

- 通道之间的数据传输(直接缓冲区)：

  ~~~java
  public void testTrans1() throws IOException{
      FileChannel inChannel = FileChannel.open(Paths.get("d:/1.mkv"), StandardOpenOption.READ);
      FileChannel outChannel = FileChannel.open(Paths.get("d:/2.mkv"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);
  
      //inChannel.transferTo(0, inChannel.size(), outChannel);
      outChannel.transferFrom(inChannel, 0, inChannel.size());
  
      inChannel.close();
      outChannel.close();
  }
  ~~~

- 使用直接缓冲区(只有ByteBuffer支持)完成文件的复制(内存映射文件)：

  ~~~java
  public void testTrans2() throws IOException{//2127-1902-1777
      long start = System.currentTimeMillis();
  
      FileChannel inChannel = FileChannel.open(Paths.get("d:/1.mkv"), StandardOpenOption.READ);
      FileChannel outChannel = FileChannel.open(Paths.get("d:/2.mkv"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);
  
      //内存映射文件
      MappedByteBuffer inMappedBuf = inChannel.map(MapMode.READ_ONLY, 0, inChannel.size());
      MappedByteBuffer outMappedBuf = outChannel.map(MapMode.READ_WRITE, 0, inChannel.size());
  
      //直接对缓冲区进行数据的读写操作
      byte[] dst = new byte[inMappedBuf.limit()];
      inMappedBuf.get(dst);
      outMappedBuf.put(dst);
  
      inChannel.close();
      outChannel.close();
  
      long end = System.currentTimeMillis();
      System.out.println("耗费时间为：" + (end - start));
  }
  ~~~

- 利用通道完成文件的复制（非直接缓冲区）:

  ~~~java
  public void test1(){//10874-10953
      long start = System.currentTimeMillis();
  
      FileInputStream fis = null;
      FileOutputStream fos = null;
      // 1. 获取通道
      FileChannel inChannel = null;
      FileChannel outChannel = null;
      try {
          fis = new FileInputStream("d:/1.mkv");
          fos = new FileOutputStream("d:/2.mkv");
  
          inChannel = fis.getChannel();
          outChannel = fos.getChannel();
  
          // 2. 分配指定大小的缓冲区
          ByteBuffer buf = ByteBuffer.allocate(1024);
  
          // 3. 将通道中的数据存入缓冲区中
          while(inChannel.read(buf) != -1){
              //切换读取数据的模式
              buf.flip(); 
              // 4. 将缓冲区中的数据写入通道中
              outChannel.write(buf);
              //清空缓冲区
              buf.clear(); 
          }
      } catch (IOException e) {
          e.printStackTrace();
      } finally {
          // 关闭资源
      }
  
      long end = System.currentTimeMillis();
      System.out.println("耗费时间为：" + (end - start));
  
  }
  ~~~


#### 分散和聚集

- 分散读取（Scattering Reads）

  是指按照缓冲区的顺序，从 Channel 中读取的数据分散到多个 Buffer 中

- 聚集写入（Gathering Writes）

  是指按照缓冲区的顺序，写入 position 和 limit 之间的数据到 Channel 

~~~java
public void testScaGath() throws IOException{
    RandomAccessFile raf1 = new RandomAccessFile("1.txt", "rw");

    //1. 获取通道
    FileChannel channel1 = raf1.getChannel();

    //2. 分配指定大小的缓冲区
    ByteBuffer buf1 = ByteBuffer.allocate(100);
    ByteBuffer buf2 = ByteBuffer.allocate(1024);

    //3. 分散读取
    ByteBuffer[] bufs = {buf1, buf2};
    channel1.read(bufs);

    for (ByteBuffer byteBuffer : bufs) {
        byteBuffer.flip();
    }

    System.out.println(new String(bufs[0].array(), 0, bufs[0].limit()));
    System.out.println("-----------------");
    System.out.println(new String(bufs[1].array(), 0, bufs[1].limit()));

    //4. 聚集写入
    RandomAccessFile raf2 = new RandomAccessFile("2.txt", "rw");
    FileChannel channel2 = raf2.getChannel();

    channel2.write(bufs);
}
~~~

#### 字符集（charset）

- 编码：字符串 -> 字节数组
- 解码：字节数组  -> 字符串

~~~java
public void testCharset() throws IOException{
    Charset cs1 = Charset.forName("GBK");

    //获取编码器
    CharsetEncoder ce = cs1.newEncoder();

    //获取解码器
    CharsetDecoder cd = cs1.newDecoder();

    CharBuffer cBuf = CharBuffer.allocate(1024);
    cBuf.put("测试数据！");
    cBuf.flip();

    //编码
    ByteBuffer bBuf = ce.encode(cBuf);
    for (int i = 0; i < 12; i++) {
        System.out.println(bBuf.get());
    }

    //解码
    bBuf.flip();
    CharBuffer cBuf2 = cd.decode(bBuf);
    System.out.println(cBuf2.toString());

    System.out.println("------------------------------------------------------");

    Charset cs2 = Charset.forName("GBK");
    bBuf.flip();
    CharBuffer cBuf3 = cs2.decode(bBuf);
    System.out.println(cBuf3.toString());
}
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

~~~java
public void testOper(){
    String str = "abcde";

    //1. 分配一个指定大小的缓冲区
    ByteBuffer buf = ByteBuffer.allocate(1024);

    System.out.println("-----------------allocate()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //2. 利用 put() 存入数据到缓冲区中
    buf.put(str.getBytes());

    System.out.println("-----------------put()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //3. 切换读取数据模式
    buf.flip();

    System.out.println("-----------------flip()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //4. 利用 get() 读取缓冲区中的数据
    byte[] dst = new byte[buf.limit()];
    buf.get(dst);
    System.out.println(new String(dst, 0, dst.length));

    System.out.println("-----------------get()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //5. rewind() : 可重复读
    buf.rewind();

    System.out.println("-----------------rewind()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    //6. clear() : 清空缓冲区. 但是缓冲区中的数据依然存在，但是处于“被遗忘”状态
    buf.clear();

    System.out.println("-----------------clear()----------------");
    System.out.println(buf.position());
    System.out.println(buf.limit());
    System.out.println(buf.capacity());

    System.out.println((char)buf.get());

}
~~~

#### 直接与非直接缓冲区

字节缓冲区要么是直接的，要么是非直接的。如果为直接字节缓冲区，则 Java 虚拟机会尽最大努力直接在此缓冲区上执行本机 I/O 操作。在每次调用基础操作系统的一个本机 I/O 操作之前（或之后），虚拟机都会尽量避免将缓冲区的内容复制到中间缓冲区中（或从中间缓冲区中复制内容）。

- 直接缓冲区

  通过 allocateDirect() 方法分配直接缓冲区，将缓冲区建立在物理内存中。可以提高效率

  ![](nio\nio05.png)

  ```java
  // 分配直接缓冲区
  ByteBuffer buf = ByteBuffer.allocateDirect(1024);
  System.out.println(buf.isDirect());
  ```

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

- 非直接缓冲区

  通过 allocate() 方法分配缓冲区，将缓冲区建立在 JVM 的内存中

  ![](nio\nio04.png)

  ~~~java
  public void testBuffer(){
      String str = "abcde";
      ByteBuffer buf = ByteBuffer.allocate(1024);
      buf.put(str.getBytes());
      buf.flip();
  
      byte[] dst = new byte[buf.limit()];
      buf.get(dst, 0, 2);
      System.out.println(new String(dst, 0, 2));
      System.out.println(buf.position());
  
      //mark() : 标记
      buf.mark();
  
      buf.get(dst, 2, 2);
      System.out.println(new String(dst, 2, 2));
      System.out.println(buf.position());
  
      //reset() : 恢复到 mark 的位置
      buf.reset();
      System.out.println(buf.position());
  
      //判断缓冲区中是否还有剩余数据
      if(buf.hasRemaining()){
          //获取缓冲区中可以操作的数量
          System.out.println(buf.remaining());
      }
  }
  ~~~


### NIO 的非阻塞式网络通信

传统的 IO 流都是阻塞式的。也就是说，当一个线程调用 read() 或 write() 时，该线程被阻塞，直到有一些数据被读取或写入，该线程在此期间不能执行其他任务。因此，在完成网络通信进行 IO 操作时，由于线程会阻塞，所以服务器端必须为每个客户端都提供一个独立的线程进行处理，当服务器端需要处理大量客户端时，性能急剧下降。

#### 非阻塞模式

Java NIO 是非阻塞模式的。当线程从某通道进行读写数据时，若没有数据可用时，该线程可以进行其他任务。线程通常将非阻塞 IO 的空闲时间用于在其他通道上执行 IO 操作，所以单独的线程可以管理多个输入和输出通道。因此，NIO 可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。

#### 选择器（Selector）

选择器（Selector） 是 SelectableChannle 对象的多路复用器，Selector 可以同时监控多个 SelectableChannel 的 IO 状况，也就是说，利用 Selector 可使一个单独的线程管理多个 Channel。Selector 是非阻塞 IO 的核心。

SelectableChannle 的结构如下图：

![](nio\nio06.png)

1. 选择器应用

   - 创建Selector：通过调用 Selector.open() 方法创建一个 Selector

     ~~~java
     Selector selector = Selector.open();
     ~~~

   - 向选择器注册通道：SelectableChannel.register(Selector sel, int ops)

     ~~~java
     // 创建一个Socket套接字
     Socket socket = new Socket(InetAddress.getByName("127.0.0.1"), 0000);
     // 获取SocketChannel
     SocketChannel channel = socket.getChannel();
     // 创建选择器
     Selector selector = Selector.open();
     // 向Selector注册Channell
     SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
     ~~~

2. SelectionKey

   表示 SelectableChannel 和 Selector 之间的注册关系。每次向选择器注册通道时就会选择一个事件(选择键)。当调用 register(Selector sel, int ops) 将通道注册选择器时，选择器对通道的监听事件类型有：

   - 读 ：SelectionKey.OP_READ （1） 
   - 写 : SelectionKey.OP_WRITE （4） 
   - 连接 : SelectionKey.OP_CONNECT （8）
   - 接收 : SelectionKey.OP_ACCEPT （16） 

   ~~~java
   // 注册监听事件
   int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE；
   ~~~

#### SocketChannel

Java NIO中的SocketChannel是一个连接到TCP网络套接字的通道。是一个可以监听新进来的TCP连接的通道，就像标准IO中的ServerSocket一样。操作步骤：

- 打开SocketChannel
- 读写数据
- 关闭SocketChannel

#### DatagramChannel

Java NIO中的DatagramChannel是一个能收发UDP包的通道。操作步骤：

- 打开DatagramChannel
- 接受/发送数据

#### 管道（Pipe）

Java NIO 管道是2个线程之间的单向数据连接。Pipe有一个source通道和一个sink通道。数据会被写到sink通道，从source通道读取。

![](nio\nio07.png)

- 向管道写数据

  ~~~java
  public void test1() throws IOException{
      String str = "测试数据";
      // 获取管道
      Pipe pipe = Pipe.open();
  
      // 将缓冲区中的数据写入管道
      Pipe.SinkChannel sinkChannel = pipe.sink();
      
      // 通过SinkChannel的write()方法写数据
      ByteBuffer buf = ByteBuffer.allocate(1024);
      buf.clear();
      buf.put(str.getBytes());
      buf.flip();
      
      while(buf.hasRemaining()) {
          sinkChannel.write(buf);
      }
  }
  ~~~

- 从管道读取数据

  读取管道的数据，需要访问source通道

  ~~~java
  // 读取缓冲区中的数据
  Pipe.SourceChannel sourceChannel = pipe.source();
  ~~~

  调用source通道的read()方法来读取数据

  ~~~java
  buf.flip();
  int len = sourceChannel.read(buf);
  System.out.println(new String(buf.array(), 0, len));
  ~~~

### NIO2

随着 JDK 7 的发布，Java对NIO进行了极大的扩展，增强了对文件处理和文件系统特性的支持，以至于我们称他们为 NIO.2。因为 NIO 提供的一些功能，NIO已经成为文件处理中越来越重要的部分。

#### Path与Paths

java.nio.file.Path 接口代表一个平台无关的平台路径，描述了目录结构中文件的位置。Paths 提供的 get() 方法用来获取 Path 对象：

~~~java
// 用于将多个字符串串连成路径。
Path get(String first, String … more)；
~~~

Path 常用方法：

~~~java
// 判断是否以 path 路径结束
boolean endsWith(String path)；
// 判断是否以 path 路径开始
boolean startsWith(String path)； 
// 判断是否是绝对路径
boolean isAbsolute()；
// 返回与调用 Path 对象关联的文件名
Path getFileName()；
// 返回的指定索引位置 idx 的路径名称
Path getName(int idx)；
// 返回Path 根目录后面元素的数量
int getNameCount()；
// 返回Path对象包含整个路径，不包含 Path 对象指定的文件路径
Path getParent()；
// 返回调用 Path 对象的根路径
Path getRoot()；
// 将相对路径解析为绝对路径
Path resolve(Path p)；
// 作为绝对路径返回调用 Path 对象
Path toAbsolutePath()；
// 返回调用 Path 对象的字符串表示形式
String toString()；
~~~

~~~java
public void testPaths(){
    Path path = Paths.get("e:/nio/hello.txt");
    System.out.println(path.getParent());
    System.out.println(path.getRoot());
    Path newPath = path.resolve("e:/hello.txt");
    System.out.println(newPath);

    Path path2 = Paths.get("1.jpg");
    Path newPath = path2.toAbsolutePath();
    System.out.println(newPath);
    System.out.println(path.toString());
    
    Path path3 = Paths.get("e:/", "nio/hello.txt");
    System.out.println(path3.endsWith("hello.txt"));
    System.out.println(path3.startsWith("e:/"));
    System.out.println(path3.isAbsolute());
    System.out.println(path3.getFileName());
    for (int i = 0; i < path3.getNameCount(); i++) {
        System.out.println(path3.getName(i));
    }
}
~~~

#### Files

java.nio.file.Files 用于操作文件或目录的工具类。

- Files常用方法：用于操作内容

  ~~~java
  // 获取与指定文件的连接，how 指定打开方式。
  SeekableByteChannel newByteChannel(Path path, OpenOption…how)；
  // 打开 path 指定的目录
  DirectoryStream newDirectoryStream(Path path)；
  // 获取 InputStream 对象
  InputStream newInputStream(Path path, OpenOption…how)；
  // 获取 OutputStream 对象
  OutputStream newOutputStream(Path path, OpenOption…how)；
  ~~~

  ~~~java
  public void testFilesContext() throws IOException{
      SeekableByteChannel newByteChannel = Files.newByteChannel(Paths.get("1.jpg"), StandardOpenOption.READ);
  
      DirectoryStream<Path> newDirectoryStream = Files.newDirectoryStream(Paths.get("e:/"));
  
      for (Path path : newDirectoryStream) {
          System.out.println(path);
      }
  }
  ~~~

- Files常用方法：用于判断

  ~~~java
  // 判断文件是否存在
  boolean exists(Path path, LinkOption … opts)；
  // 判断是否是目录
  boolean isDirectory(Path path, LinkOption … opts)；
  // 判断是否是可执行文件
  boolean isExecutable(Path path)；
  // 判断是否是隐藏文件
  boolean isHidden(Path path)；
  // 判断文件是否可读
  boolean isReadable(Path path)；
  // 判断文件是否可写
  boolean isWritable(Path path)； 
  // 判断文件是否不存在
  boolean notExists(Path path, LinkOption … opts)；
  // 获取与 path 指定的文件相关联的属性。
  public static <A extends BasicFileAttributes> A readAttributes(Path path,Class<A> type,LinkOption... options)；
  ~~~

  ~~~java
  public void testFilesJudge() throws IOException{
      Path path = Paths.get("e:/nio/hello7.txt");
      System.out.println(Files.exists(path, LinkOption.NOFOLLOW_LINKS));
  
      BasicFileAttributes readAttributes = Files.readAttributes(path, BasicFileAttributes.class, LinkOption.NOFOLLOW_LINKS);
      System.out.println(readAttributes.creationTime());
      System.out.println(readAttributes.lastModifiedTime());
  
      DosFileAttributeView fileAttributeView = Files.getFileAttributeView(path, DosFileAttributeView.class, LinkOption.NOFOLLOW_LINKS);
  
      fileAttributeView.setHidden(false);
  }
  ~~~

- Files常用方法：

  ~~~java
  // 文件的复制
  Path copy(Path src, Path dest, CopyOption … how)；
  // 创建一个目录
  Path createDirectory(Path path, FileAttribute<?> … attr)；
  // 创建一个文件
  Path createFile(Path path, FileAttribute<?> … arr)；
  // 删除一个文件
  void delete(Path path)；
  // 将 src 移动到 dest 位置
  Path move(Path src, Path dest, CopyOption…how)；
  // 返回 path 指定文件的大小
  long size(Path path)；
  ~~~

  ~~~java
  public void testFiles() throws IOException{
      Path path1 = Paths.get("e:/nio/hello2.txt");
      Path path2 = Paths.get("e:/nio/hello7.txt");
      
      System.out.println(Files.size(path2));
      
      Files.copy(path1, path2, StandardCopyOption.REPLACE_EXISTING);
      Files.move(path1, path2, StandardCopyOption.ATOMIC_MOVE);
      
      Path dir = Paths.get("e:/nio/nio2");
      Files.createDirectory(dir);
  
      Path file = Paths.get("e:/nio/nio2/hello3.txt");
      Files.createFile(file);
  
      Files.deleteIfExists(file);
  }
  ~~~

#### 自动资源管理

Java 7 增加了一个新特性，该特性提供了另外一种管理资源的方式，这种方式能自动关闭文 件。这个特性有时被称为自动资源管理 (Automatic Resource Management, ARM)， 该特性以 try 语句的扩展版为基础。自动资源管理主要用于，当不再需要文件（或其他资源）时，可以防止无意中忘记释放它们。自动资源管理基于 try 语句的扩展形式：

~~~java
try(需要关闭的资源声明) {
	//可能发生异常的语句
} catch(异常类型 变量名) {
	//异常的处理语句
} finally {
	//一定执行的语句
}
~~~

当 try 代码块结束时，自动释放资源。因此不需要显示的调用 close() 方法。该形式也称为“带资源的 try 语句”。

注意：

- try 语句中声明的资源被隐式声明为 final ，资源的作用局限于带资源的 try 语句
- 可以在一条 try 语句中管理多个资源，每个资源以“;” 隔开即可。
-  需要关闭的资源，必须实现了 AutoCloseable 接口或其自接口 Closeable

~~~java
public void testAutoClose(){
    try(FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);
        FileChannel outChannel = FileChannel.open(Paths.get("2.jpg"), StandardOpenOption.WRITE, StandardOpenOption.CREATE)){

        ByteBuffer buf = ByteBuffer.allocate(1024);
        inChannel.read(buf);

    }catch(IOException e){

    }
}
~~~

### 网络通信

使用 NIO 完成网络通信的三个核心：

1. 通道（Channel）：负责连接
2. 缓冲区（Buffer）：负责数据的存取
3. 选择器（Selector）：是 SelectableChannel 的多路复用器。用于监控 SelectableChannel 的 IO 状况

#### 阻塞式

客户端

~~~java
public void client() throws IOException{
    //1. 获取通道
    SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 0000));

    FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);

    //2. 分配指定大小的缓冲区
    ByteBuffer buf = ByteBuffer.allocate(1024);

    //3. 读取本地文件，并发送到服务端
    while(inChannel.read(buf) != -1){
        buf.flip();
        sChannel.write(buf);
        buf.clear();
    }

    //接收服务端的反馈
    sChannel.shutdownOutput();
    int len = 0;
    while((len = sChannel.read(buf)) != -1){
        buf.flip();
        System.out.println(new String(buf.array(), 0, len));
        buf.clear();
    }
    
    //4. 关闭通道
    inChannel.close();
    sChannel.close();
}
~~~

服务端

~~~java
public void server() throws IOException{
    //1. 获取通道
    ServerSocketChannel ssChannel = ServerSocketChannel.open();

    FileChannel outChannel = FileChannel.open(Paths.get("2.jpg"), StandardOpenOption.WRITE, StandardOpenOption.CREATE);

    //2. 绑定连接
    ssChannel.bind(new InetSocketAddress(0000));

    //3. 获取客户端连接的通道
    SocketChannel sChannel = ssChannel.accept();

    //4. 分配指定大小的缓冲区
    ByteBuffer buf = ByteBuffer.allocate(1024);

    //5. 接收客户端的数据，并保存到本地
    while(sChannel.read(buf) != -1){
        buf.flip();
        outChannel.write(buf);
        buf.clear();
    }
    
    // 发送反馈给客户端
    buf.put("服务端接收数据成功".getBytes());
    buf.flip();
    sChannel.write(buf);

    //6. 关闭通道
    sChannel.close();
    outChannel.close();
    ssChannel.close();

}
~~~

#### 非阻塞式

##### TCP

客户端

~~~java
public void client() throws IOException{
    //1. 获取通道
    SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 0000));

    //2. 切换非阻塞模式
    sChannel.configureBlocking(false);

    //3. 分配指定大小的缓冲区
    ByteBuffer buf = ByteBuffer.allocate(1024);

    //4. 发送数据给服务端
    Scanner scan = new Scanner(System.in);

    while(scan.hasNext()){
        String str = scan.next();
        buf.put((new Date().toString() + "\n" + str).getBytes());
        buf.flip();
        sChannel.write(buf);
        buf.clear();
    }

    //5. 关闭通道
    sChannel.close();
}
~~~

服务端

~~~java
public void server() throws IOException{
    //1. 获取通道
    ServerSocketChannel ssChannel = ServerSocketChannel.open();

    //2. 切换非阻塞模式
    ssChannel.configureBlocking(false);

    //3. 绑定连接
    ssChannel.bind(new InetSocketAddress(0000));

    //4. 获取选择器
    Selector selector = Selector.open();

    //5. 将通道注册到选择器上, 并且指定“监听接收事件”
    ssChannel.register(selector, SelectionKey.OP_ACCEPT);

    //6. 轮询式的获取选择器上已经“准备就绪”的事件
    while(selector.select() > 0){

        //7. 获取当前选择器中所有注册的“选择键(已就绪的监听事件)”
        Iterator<SelectionKey> it = selector.selectedKeys().iterator();

        while(it.hasNext()){
            //8. 获取准备“就绪”的是事件
            SelectionKey sk = it.next();

            //9. 判断具体是什么事件准备就绪
            if(sk.isAcceptable()){
                //10. 若“接收就绪”，获取客户端连接
                SocketChannel sChannel = ssChannel.accept();

                //11. 切换非阻塞模式
                sChannel.configureBlocking(false);

                //12. 将该通道注册到选择器上
                sChannel.register(selector, SelectionKey.OP_READ);
            }else if(sk.isReadable()){
                //13. 获取当前选择器上“读就绪”状态的通道
                SocketChannel sChannel = (SocketChannel) sk.channel();

                //14. 读取数据
                ByteBuffer buf = ByteBuffer.allocate(1024);

                int len = 0;
                while((len = sChannel.read(buf)) > 0 ){
                    buf.flip();
                    System.out.println(new String(buf.array(), 0, len));
                    buf.clear();
                }
            }

            //15. 取消选择键 SelectionKey
            it.remove();
        }
    }
}
~~~

##### UDP

客户端

~~~java
public void send() throws IOException{
    DatagramChannel dc = DatagramChannel.open();

    dc.configureBlocking(false);

    ByteBuffer buf = ByteBuffer.allocate(1024);

    Scanner scan = new Scanner(System.in);

    while(scan.hasNext()){
        String str = scan.next();
        buf.put((new Date().toString() + ":\n" + str).getBytes());
        buf.flip();
        dc.send(buf, new InetSocketAddress("127.0.0.1", 9898));
        buf.clear();
    }

    dc.close();
}
~~~

服务端

~~~java
public void receive() throws IOException{
    DatagramChannel dc = DatagramChannel.open();

    dc.configureBlocking(false);

    dc.bind(new InetSocketAddress(9898));

    Selector selector = Selector.open();

    dc.register(selector, SelectionKey.OP_READ);

    while(selector.select() > 0){
        Iterator<SelectionKey> it = selector.selectedKeys().iterator();

        while(it.hasNext()){
            SelectionKey sk = it.next();

            if(sk.isReadable()){
                ByteBuffer buf = ByteBuffer.allocate(1024);

                dc.receive(buf);
                buf.flip();
                System.out.println(new String(buf.array(), 0, buf.limit()));
                buf.clear();
            }
        }

        it.remove();
    }
}
~~~





