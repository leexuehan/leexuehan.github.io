---
title: IO vs NIO
tags:
  - Java
date: 2015-08-21 11:56:12
categories: 技术
---

昨天面试问到了有关Java NIO的问题，没有答上来。于是，在网上看到了一篇很有用的系列文章讲Java IO的，浅显易懂。后面的备注里有该系列文章的链接。内容不算很长，需要两个小时肯定看完了，将该系列文章看完之后，我又参看了一些其他的资料，形成了一点自己不成熟的理解，将之记录下来，算是我自己学到的一点东西吧！有什么问题，希望大神能够不吝赐教。

## IO 的分类？
我自己的观点,IO大致可以分为本地IO（磁盘IO）和网络IO。所谓的本地IO，即本地的应用程序从磁盘里面读写数据，不涉及网络的概念；而网络IO则是发送端发送数据（写），接收端接收数据（读），当然，也不一定是发送端只能发送数据，接收端只能接收数据，如果是基于TCP/IP协议的话，可以一边发一边收，因为它是全双工的协议。

## 磁盘IO
我们知道应用程序访问物理设备只能通过系统调用的方式来工作的，读写分别用read()和write()来操作。

而系统调用就存在**用户空间**和**内核空间**的问题。

为什么要分成这两个空间呢？因为操作系统要保证应用程序不能随便修改内核的数据，是为了安全起见。不然，你随便写一个能修改内核数据的app，有可能导致系统崩溃。

分开后，就涉及到了一个复制的问题。

下面介绍几种常用的访问文件的方式：

### 标准访问文件的方式

以读取数据为例：

应用程序---系统调用-read()--->应用缓存（用户地址空间）--->高速缓存（内核地址空间）---->物理磁盘

写入数据的过程也类似。

因为用户空间和内核空间都有缓存，所以无论是读入数据还是写入数据，都要先经过缓存这一步。比如读的时候，先要检查缓存是否有，没有才从磁盘中读；写的时候，要先写到缓存中，如果不是显式地调用sync同步命令，什么时候写入磁盘就要看操作系统了。

缺点：隔离用户空间和地址空间是为了保证系统安全，但也增加了复制的麻烦，比如写入数据的时候，先要写入应用缓存中，然后再复制到内核空间里面的告诉缓存中，最后才是磁盘。
		
### 直接IO方式

以读取数据为例：

应用程序---系统调用-read()--->磁盘

很明显，少了一个经过内核空间的步骤。

这种访问方式的应用场合：

比如，数据库管理系统，系统明确知道应该缓存哪些数据，应该失效哪些数据，还可以将一些数据做预加载，这样可以加速数据的访问效率；但是如果由操作系统来负责缓存，则很难做到。

缺点：访问的数据不在应用缓存中，直接从磁盘加载的速度非常缓慢。

### 同步访问文件的方式

数据的读取和写入同步，只有当数据被成功写入到磁盘时才返回给应用程序成功标志。

### 异步访问文件的方式

访问数据的线程发出请求之后，线程接着去处理其他事情，而不是阻塞等待。

### 内存映射方式

内存中的某一块区域与磁盘中的文件关联起来的，当要访问内存中的一段数据时，转换为访问文件的某一段数据，这种方式同样可以减少数据从内核空间缓存到用户空间缓存的数据复制操作。**因为这样两个空间的数据是共享的**。



### Java访问磁盘文件的接口

1.Java里面的四组IO接口：

基于**字节**操作的IO接口：InputStream 和 OutputStream

基于**字符**操作的IO接口：Writer和 Reader

<font color="red">此处字符和字节一定要搞清楚，上次电话面试就弄错了，汗。。。。。</font>

<font color="blue">字节IO流向字符IO流转换只需要一个装饰类即可，关于此处的细节，会在装饰器模式的Java IO流部分讲到。。</font>

基于**磁盘**操作的IO接口：File

基于**网络**操作的IO接口： Socket

2.Java访问磁盘文件的方式

需要知道一句话：**数据在磁盘中的唯一最小描述就是文件，也就是说上层应用程序只能通过文件来操作磁盘上的数据，文件也是操作系统与磁盘驱动交互的最小单元。**

下面是一个Java从磁盘中读取文件的过程：

> start--->传入文件路径--->创建File对象来标识此文件---->创建StreamDecoder解码对象(由该对象将输入流解码为字符流还是字节流)--->创建FileInputStream对象(<font color="red">
> 这个才是真正读取文件的对象</font>，同时创建磁盘文件的文件描述符 FileDescriptor,此对象负责控制这个磁盘文件)--->over

### Java Socket

在Java里面通过Socket实现通信的过程：

1.建立通信链路。

客户端：

创建一个Socket实例，同时操作系统将为这个实例分配一个没有被使用的本地端口号，并创建一个包含本地地址，远程地址，端口号的socket数据结构。这个数据结构将一直保存在系统中直到这个连接关闭。

需要注意的是：**在创建的Socket实例的构造函数正确返回之前，需要进行三次握手，握手完成，Socket实例对象创建完成，否则报错。**

服务端：

服务端创建一个ServerSocket实例（只要端口号没有被占用，实例都会被创建成功。）同时操作系统也会为ServerSocket实例创建一个底层数据结构，这个数据结构包含指定监听端口的端口号，和监听地址的通配符，通常为“*”，即监听所有地址。

然后调用accept方法进入阻塞状态，等待客户端请求。当一个新的请求时，将为这个连接创建一个新的套接字数据结构（此数据结构包括请求源地址和端口），之后这个新的数据结构会关联到ServerSocket实例的一个未完成的连接数据结构列表中。

再次注意：与上面客户端创建Socket实例过程一样，必须在三次握手之后才能创建真正的实例，然后将关联的数据结构从未完成列表移到已完成列表中。

2.传输数据

通过第一步，创建了两个Socket实例，每个实例都有一个InputStream和OutputStream。这两个实例都可以通过getInputStream()方法和getOutputStream()方法得到。

创建Socket对象时，操作系统将为InputStream和OutputStream分配一定大小的缓存区，此缓存区用于数据的读取和写入。

写入端将数据写到OutputStream对应的SendQ队列中，当队列满时，开始向另一端InputStream的RecvQ队列中发送。如果接收端的RecvQ已满，则输入端的write方法阻塞，直到接收端的Recv队列有空闲，然后开始继续发送。

需要注意的是：如果两边同时开始发送数据怎么办？

这时候，就需要一个协调机制，来防止发生死锁。这个机制暂且不提，有机会再完成更新。

但是NIO则没有这种死锁情况的发生。

## NIO

### Java NIO是什么？

我们所熟知的IO方式就是通过InputStream/OutputStream, InputStreamReader/OutputStreamWriter的方式从磁盘里面读取数据，或者向磁盘写数据。
关于本地磁盘读取文件前面已有详细描述，此处不表。

Java NIO用一个老外的话来说就是：


> In the standard IO API you work with byte streams and character streams. In NIO you work with channels and buffers.Data is always read from a channel into  a buffer, or written from a buffer to a channel.

NIO,即 Non-blocking IO,非阻塞式的IO，上面讲的IO都是阻塞式的。什么是阻塞式的？就是你如果要从磁盘上读取一个文件，在读的过程中你不能干其他事，只有等待它读取完毕，你才能干后面的事情；而非阻塞式，显然，就可以在读取文件的过程中，做别的事情。

NIO 采用**内存映射**的方式来处理输入和输出。它将文件或者文件的一段区域映射到内存中，这样就可以像访问内存一样来访问文件了，类似于操作系统上的虚拟内存的概念，通过这种方式来进行输入和输出要比传统的输入和输出要快很多。

## NIO的核心概念

上面提到两个概念，channel, buffer，还有一个selector。这三个构成了NIO的核心。

在NIO里面所有的IO都从一个channel开始，channel有点像Stream，但是还是有很多细微的区别的，在下面会被提到。数据可以从channel读到一个buffer中，也可以从buffer写入到channel里面。

在Java NIO里面有以下四种channel：

		FileChannel---对文件进行读写数据
		
		DatagramChannel-----通过UDP读写数据
		
		SocketChannel------通过TCP读写数据
		
		ServerSocketChannel----服务端监听TCP连接

这些channel涵盖了UDP+TCP，网络IO和文件IO。

Java NIO里面的buffer：

		ByteBuffer
		
		CharBuffer
		
		DoubleBuffer
		
		FloatBuffer
		
		IntBuffer
		
		LongBuffer
		
		ShortBuffer

也就是每种基本类型都有相应的Buffer。

一个Selector允许一个单线程操作多个Channel。但是前提是**你必须在相应的Channel上注册了这个Selector。**然后你调用select方法，如果在channel上没有一个事件是准备好，那么这个方法将被阻塞。


NIO Channel和Streams的区别：

1.你可以对Channel进行读和写，但是Stream通常只能读，或者只能写。

2.Channel可以读写异步进行。

3.Channel总是从Buffer中读数据，或者向Buffer中写数据。

4.最大的区别在于它提供了一个 map 方法，可以将“一块数据”映射到内存中。

一个使用Channel的例子：

	    RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
	    FileChannel inChannel = aFile.getChannel();
	
	    ByteBuffer buf = ByteBuffer.allocate(48);
	
	    int bytesRead = inChannel.read(buf);
	    while (bytesRead != -1) {
	
	      System.out.println("Read " + bytesRead);
	      buf.flip();
	
	      while(buf.hasRemaining()){
	          System.out.print((char) buf.get());
	      }
	
	      buf.clear();
	      bytesRead = inChannel.read(buf);
	    }
	    aFile.close();

通过Buffer读写数据一般要遵循下面四步：

> 写入Buffer
> 
> 调用buffer.flip()
> 
> 将Buffer里面的数据读尽
> 
> 调用方法buffer.clear()/buffer.compact()
> 

当你将数据写入Buffer时，Buffer将记录你写入数据的数量，如果你想从Buffer中读取data的话，则**需要调用flip()方法将Buffer从写入模式转换成读取模式。**当你将数据读完时，需要调用clear()方法将Buffer清理干净（或者也可以用compact方法，它只将你读取数据的那一部分清理干净）。


Buffer有3个参数：Capacity,Position,Limit.

Capacity：顾名思义，Buffer的容量。

Position：当你向Buffer写入数据的时候，需要一个位置指示，这个指示就是Position，初始化为0.

Limit：在写模式下，意思是你最多可以向Buffer写入多少数据；在读模式下，意思是你可以最多从Buffer中读取多少数据。

Buffer 的主要作用是装入数据，然后输出数据。当装入数据结束后，调用其 flip 方法，该方法将 limit 设置为 Position 所在位置，并将 Position 设为 0。当输出结束后，再调用 clear 方法，为再次向 Buffer 装入数据做好准备。

怎么分配Buffer？

		ByteBuffer buf = ByteBuffer.allocate(48);//分配48字节的Buffer

		CharBuffer buf = CharBuffer.allocate(1024);//分配1024字节的Buffer

Channel：

程序不能直接访问 Channel中的数据，无论是读还是写。Channel 只能与 Buffer 进行交互。

Channel 的种类很多，此处略去。

所有的Channel 都不应该通过构造器来直接创建，而是通过传统的节点 InputStream、OutputStream的getChannel 方法来返回对应的 Channel， 不同节点流获得的 Channel 不一样。

Channel 最常用的 3 类方法：

map()：将 Channel 对应的部分或者全部数据映射成为 ByteBuffer；

read() 、write(): 用于从Buffer中读写数据。

下面讲讲Selector：

使用Selector的一大优点是你仅仅利用一个单线程就可以操纵多个Channel。**因为线程的上下文切换是很消耗资源的。**所以线程当然越少越好。

创建一个Selector：

	Selector selector = Selector.open();

向Channel注册一个Selector：

	channel.configureBlocking(false);//注意这里先要将Channel配置成为非阻塞的
	SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
register的第二个参数，意思是你对这个Channel感兴趣的事件是什么或者通过Selector监听什么事件。

下面是监听的不同事件的种类和相应的表示：

Connect----SelectionKey.OP_CONNECT

Accept-----SelectionKey.OP_ACCEPT

Read-------SelectionKey.OP_READ

Write------SelectionKey.OP_WRITE


## NIO VS IO

上面讲的其实还不是很细致，有必要找一本Java网络编程的书来看看。这里只讲了一些大概，知道具体是怎么回事，但是更多的细节，无奈水平所限，只能等到将来完善了。

下面对NIO和 IO进行一些比较，进而让我们知道，什么时候用IO好一些什么时候用NIO好一点。

NIO和IO最主要的区别就是：NIO是面向Stream的，而NIO是面向Buffer的。这句话是什么意思呢？

面向Stream，意思是你一次从一个Stream里面读取一个或者多个Bytes，你对这些Bytes做什么，将由你来决定。而且这些Bytes没有缓存在任何地方，所以你不能将这些data随意前后移动。

面向Buffer，意思是你可以将数据读取到一个Buffer里面之后进行处理。如果有必要的话，还可以将这些数据前后移动，这给你带来很大的灵活性。**但是，你也不许检查来确保Buffer里面是否已经包含了所有你想要的数据，同时再次读取数据的时候，不会覆盖你尚未处理的数据。**


NIO是非阻塞式的，而IO是阻塞式的，区别前面已经介绍过。

NIO和IO各有什么适合的应用场景呢？

如果你需要同时处理成千上万的打开的连接，但是每个连接可能只发送一小部分数据，例如一个聊天软件的服务器，NIO会是一个不错的选择；如果你在带宽非常高的情况下，只有数量很小的连接，并且一次发送大量的数据，传统的IO或许更适合。



## 备注
###### 这里有一个链接，是一个系列教程，是老外写的讲NIO的。[http://tutorials.jenkov.com/java-nio/index.html](http://tutorials.jenkov.com/java-nio/index.html)
还有一个网站专门讲Java并发编程的网站[http://ifeve.com/category/talk-concurrent/](http://ifeve.com/category/talk-concurrent/)