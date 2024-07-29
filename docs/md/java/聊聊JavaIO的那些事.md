> # 聊聊Java IO的那些事

#NIO #BIO #AIO

|      | BIO  |     NIO     |  AIO  |
| :--: | :--: | :---------: | :---: |
| IO模型 | 同步阻塞 | 同步非阻塞（多路复用） | 异步非阻塞 |
| 编程难度 |  简单  |     复杂      |  复杂   |
| 可靠性  |  差   |      好      |   好   |
| 吞吐量  |  低   |      高      |   高   |
# 重要概念
`阻塞 IO` 和`非阻塞 IO`

这两个概念是`程序级别`的。主要描述是程序请求操作系统 IO 操作之后，如果 IO 资源没有准备好，那么程序如何处理问题：前者等待，后者继续执行（并且使用线程一直轮询，知道有 IO 资源准备好）

`同步 IO `和`非同步 IO`

这两个概念是`操作系统级别`的。主要描述的是操作系统在收到程序请求 IO 操作后，如果 IO 资源没有准备好，该如何相应程序的问题：前者不响应，后者返回一个标记，当 IO 资源准备好之后，在用事件机制返回给程序。

# 一、BIO（Blocking I/O）
同步并阻塞（传统阻塞性），应用程序中进程在发起 IO 调用后至内核执行 IO 操作返回结果之前，若发起系统调用的线程一直处于等待状态，则此次 IO 操作为阻塞 IO。阻塞 IO 简称 BIO，Blocking IO。

以前大多数网络通信方式都是阻塞模式，即：

- 客户端向服务器端发送请求后，客户端会一直等待（不会再做其他事情），直到服务器端返回结果或者网络出现问题。
- 服务端同样的，当在处理某个客户端 A 发来的请求是，另一个客户端 B 发来的请求会等待，直到服务器端的这个处理线程完成上个处理。

![[acea8af4268c8d552741ccebcb2d34ec_MD5.png]](img/acea8af4268c8d552741ccebcb2d34ec_MD5.png)
## 1. BIO 编程流程

1. 服务器启动一个 ServerSocket。
2. 客户端启动 Socket 对服务器进行通信，默认情况下服务器端需要对每个客户建立一个线程与之通讯。
3. 客户端发出请求后，先咨询服务器是否有线程响应，如果没有则会等待，或者被拒绝。
4. 如果有响应，客户端线程会等待请求结束后，再继续执行。

具体可以看一下代码：

```java
package com.atguigu.bio;

import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BIOServer {
    
    public static void main(String[] args) throws Exception {
        //线程池机制
        //思路
        //1. 创建一个线程池
        //2. 如果有客户端连接，就创建一个线程，与之通讯(单独写一个方法)
        ExecutorService newCachedThreadPool = Executors.newCachedThreadPool();
        //创建ServerSocket
        ServerSocket serverSocket = new ServerSocket(6666);
        System.out.println("服务器启动了");
        while (true) {
            System.out.println("线程信息id = " + Thread.currentThread().getId() + "名字 = " + Thread.currentThread().getName());
            //监听，等待客户端连接
            System.out.println("等待连接....");
            //会阻塞在accept()
            final Socket socket = serverSocket.accept();
            System.out.println("连接到一个客户端");
            //就创建一个线程，与之通讯(单独写一个方法)
            newCachedThreadPool.execute(new Runnable() {
                public void run() {//我们重写
                    //可以和客户端通讯
                    handler(socket);
                }
            });
        }
    }
    
    //编写一个handler方法，和客户端通讯
    public static void handler(Socket socket) {
        try {
            System.out.println("线程信息id = " + Thread.currentThread().getId() + "名字 = " + Thread.currentThread().getName());
            byte[] bytes = new byte[1024];
            //通过socket获取输入流
            InputStream inputStream = socket.getInputStream();
            //循环的读取客户端发送的数据
            while (true) {
                System.out.println("线程信息id = " + Thread.currentThread().getId() + "名字 = " + Thread.currentThread().getName());
                System.out.println("read....");
                int read = inputStream.read(bytes);
                if (read != -1) {
                    System.out.println(new String(bytes, 0, read));//输出客户端发送的数据
                } else {
                    break;
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            System.out.println("关闭和client的连接");
            try {
                socket.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

## 2. 问题分析
传统的 IO 模型，其主要是一个 Server 对接 N 个客户端，在客户端连接之后，为每个客户端分配一个子线程。如图所示：

![[0fb77c535fcbf8f4264b6eff292fd210_MD5.png|1025]](img/0fb77c535fcbf8f4264b6eff292fd210_MD5.png)

从图中可以看出，传统 IO 的特点在于：
- 每个客户端连接到达时，服务端会分配一个线程给该客户端，该线程处理包括读取数据，解码，业务计算，编码，以及发送数据整个过程
- 同一时刻，服务端的吞吐量与服务器所提供的线程数量呈线性关系的。

如果并发量不大，运行没有问题，但是如果海量并发时候，就会出现问题：
1. 每次请求都要创建独立的线程，与对应的客户端进行数据的Read，业务处理，数据Write、
2. 当并发数较大时，需要创建大量线程处理连接，资源占用较大。
3. 连接建立后，如果当前线程展示没有数据可读，则线程就阻塞在Read操作上，造成线程资源浪费。

## 3. 多线程方式 - 伪异步方式
上述说的情况只是服务器只有一个线程的情况，那么如果引入多线程是不是可以解决这个问题：
- 当服务器收到客户端 X 的请求后，（读取到所有的请求数据后）将这个请求送入到一个独立线程进行处理，然后主线程继续接收客户端 Y 的请求。
- 客户端侧，也可以用一个子线程和服务器端进行通信。这样客户端主线程的其他工作不受影响，当服务器有响应信息时候再有这个子线程通过 `监听模式/观察模式`（等其他设计模式）通知主线程。

![[ed84ace61748c9dbdaf3f9718f21ff21_MD5.png]](img/ed84ace61748c9dbdaf3f9718f21ff21_MD5.png)

但是多线程解决这个问题有局限性：

- 操作系统通知accept() 的方式还是单个，即：服务器收到数据报文之后的“业务处理过程”可以多线程，但是报文的接收还是需要一个个来
- 在操作系统中，线程是有限的。线程越多，CPU 切换所需时间也越长，用来处理真正业务的需求也就越少。
- 创建线程需要较大的资源消耗。JVM 创建一个线程，即使不进行任何工作，也需要分配一个堆栈空间（128k）。
- 如果程序中使用了大量的长连接，线程是不会关闭的，资源消耗更容易时空。

为啥 `serverSocket. accept()`会出现阻塞？

是因为 Java 通过 JNI 调用的系统层面的 `accept0()` 方法，`accept0()` 规定如果发现套件子从指定的端口来，就会等待。

# 二、NIO（non-Blocking I/O）

Java NIO：同步非阻塞，服务器实现模式为一个线程处理多个请求（连接），即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮训到连接有 I/O 请求就进行处理。

## 1. 基本介绍

1. `三大核心：` Channel（通道）、Buffer（缓冲区）、Selector（选择器）。
2. `面向缓冲区，或者是面向块编程。` 数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动，这就增加了处理过程中的林火星，使用它可以提供非阻塞式的高伸缩弹性网络。
3. `非阻塞模式`, 使一个线程从某个通道发送请求或者读取数据，但是他仅能得到目前可用的数据，如果目前没有数据可用，就什么都不会获取，而不是保持线程阻塞。所以知道数据变得可读取之前，该线程还可以去做其他实行。
4. `Channel 和 Buffer 一一对应`。
5. `一个线程只有一个 Selector，一个线程对应对个 Channel（连接）`。
6. 程序切换到哪个 Channel 是由事件决定，Event 就是个重要概念。
7. Selector 会根据不同的事件，在各个通道上切换。
8. `Buffer 是一个内存块，底层就是一个数组`。
9. 数据读写都是通过 Buffer，区别于 BIO 的输入输出流，且`双向`，需要 `flip 方法切换 Channel 是双向的`。

## 2. 编程原理

1. 当客户端连接时，会通过 ServerSocketChannel 得到 SocketChannel。
2. Selector 进行减轻 select 方法，返回有事件发生的通道个数。
3. 将 SocketChannel 注册到 Selector 上（register(Selector selector, int ops)），一个 Selector 可以注册多个 SocketChannel。
4. 注册后返回 SelectionKey，会和该 Selector 关联（集合）。
5. 当有事件发生时，进一步得到各个 SelectionKey。
6. 通过channel () 方法，用 SelectionKey 反向获取 SocketChannel。
7. 可以通过得到的 channel，完成业务处理。

## 3. 缓冲区（Buffer）

###  Buffer 类及其子类

BytBuffer,字节数据；ShortBuffer,字符串数据；CharBuffer,字符数据；IntBuffer,整数；

LongBuffer,长整数；DoubleBuffer,小数；FloatBuffer,小数

### Buffer 属性和方法

Buffer 类提供了 4 个属性来提供数据元素信息：

`capacity（容量）`：缓存区的最大容量

`Limit（终点）`：缓存区最大可操作位置

`Position（位置）` ：缓存区当前在操作的位置

`Mark（标记）`：标记位置


```java
public abstract class Buffer{
	public final int capacity();
	public final int position();
	public final Buffer position(int newPosition);
	public final int limit();
	public final Buffer limit(int newLimit);
	
}
//其中比较常用的就是ByteBuffer（二进制数据），该类主要有以下方法
public abstract class ByteBuffer(){
	public static ByteBuffer allocateDirect(int capacity);//直接创建缓冲区
	public static ByteBuffer allocate(int capacity);//设置缓冲区的初始容量
	public static ByteBuffer wrap(byte[] array);//把一个数组放入到缓冲区使用
	//构造初始化位置offset和上界length的缓冲区
	public static ByteBuffer wrap(byte[] array,int offset,int length);
	//缓冲区读取相关API
	public abstract byte get();//从当前位置position上get，get之后，positon会+1
	public abstract byte get(int index);//从绝对位置获取
	public abstract ByteBuffer put(byte b);//当前位置上put，put之后，position会+1
	public abstract ByteBuffer put(int index,byte b);//从绝对位置put	
}
```

状态变量的改变过程举例:

① 新建一个大小为 8 个字节的缓冲区，此时 position 为 0，而 limit = capacity = 8。capacity 变量不会改变，下面的讨论会忽略它。

![[a8f0d4502adb087892e11866bdac7d57_MD5.png|1000]](img/a8f0d4502adb087892e11866bdac7d57_MD5.png)

② 从输入通道中读取 5 个字节数据写入缓冲区中，此时 position 移动设置为 5，limit 保持不变。

![[5ee1af7a6012fd34b62704d5b2867320_MD5.png|1000]](img/5ee1af7a6012fd34b62704d5b2867320_MD5.png)

③ 在将缓冲区的数据写到输出通道之前，需要先调用 flip() 方法，这个方法将 limit 设置为当前 position，并将 position 设置为 0。

![[5cd2995739f1b0e1f4b355a2471c38aa_MD5.png|1025]](img/5cd2995739f1b0e1f4b355a2471c38aa_MD5.png)

④ 从缓冲区中取 4 个字节到输出缓冲中，此时 position 设为 4。

![[0d87f8ba4e770fbdcd6c6fc61fb84862_MD5.png|1025]](img/0d87f8ba4e770fbdcd6c6fc61fb84862_MD5.png)

⑤ 最后需要调用 clear() 方法来清空缓冲区，此时 position 和 limit 都被设置为最初位置。

![[f6ef08bdd8b4ff67a419bfe9b7dbc0f2_MD5.png|1025]](img/f6ef08bdd8b4ff67a419bfe9b7dbc0f2_MD5.png)

## 4. 通道（Channel）

通道类似流，但是有如下区别：
- 通道可以同时读写，而流只能读或写
- 以实现异步读写数据
- 以从缓冲区读数据，也可以写数据到缓冲区

### 通道分类

Channel 在 NIO 中是一个接口 `public interface Channle extends Closeable{}`

常用的 Channel 类有： 
- FileChannel：用于文件的数据读写
- DatagramChannel：用于 UDP 的数据读写
- ServerSocketChannel：可以监听新来的连接，对每一个新进来的连接都会创建一个 SocketChannel。只有通过这个通道，应用程序才能箱操作系统注册支持“多路复用 IO”的端口减轻。支持 TCP 和 UDP 协议
- SocketChannel：TCP Socket 套接字的监听通道，用于 TCP 的数据读写

其他的通道包括：

![[94530647a3be7da4d2de055fff8bacaf_MD5.png|1025]](img/94530647a3be7da4d2de055fff8bacaf_MD5.png)

### FileChannel 类

对本地文件进行 IO 操作，常用方法及实例应用：

```java
//从通道读取数据并放到缓冲区内
public int read(ByteBuffer content);
//从缓冲区写数据到通道中
public int write(ByteBuffer content);
//从目标通道中复制数据到当前通道内
public long transferFrom(ReadableByteChannel src,long position,long count);
//把数据从当前通道复制到目标通道
public long transferTo(long position,long count,WritabelByteChannel target);

```
#### 1 . 写入文件，使用之前 ByteBuffer 和 FileChannel 类

``` java
//使用之前ByteBuffer和FileChannel类，写入文件

import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class NIOFileChannel{
	public static void main(String[] args) throws Exception{
		String str = "hello,world";
		//创一个输出流 -> channel
		FileOutputStream stream = new FileOutputStream("d:\\file.txt");
		//通过 stream 获取对应的 FileChannel
		//这个 fileChannel 真实类型是 FileChannelImpl
		FileChannel fileChannel = stream.getChannel();

		//创建一个缓冲区 ByteBuffer
		ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
		//将 str 放入到缓冲区
		byteBuffer.put(str.getBytes());
		//对 byteBuffer 记性 flip
		byteBuffer.flip();

		//将 byteBuffer 写入到 fileChannel
		fileChannel.write(byteBuffer);
		fileOutputStream.close();
	}

}
```

#### 2 . 读取文件数据并展示，使用之前 ByteBuffer 和 FileChannel 类


```java
//读取本地文件
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class NIOFileChannel{
	public static void main(String[] args) throws Exception{
		//创一个输出流 -> channel
		File file = new File("d:\\file.txt");
		FileOutputStream stream = new FileOutputStream(file);
		//通过 stream 获取对应的 FileChannel
		//这个 fileChannel 真实类型是 FileChannelImpl
		FileChannel fileChannel = stream.getChannel();

		//创建一个缓冲区 ByteBuffer
		ByteBuffer byteBuffer = ByteBuffer.allocate((int)file.length());

		//将 byteBuffer 写入到 fileChannel
		fileChannel.read(byteBuffer);
		//将 byteBuffer的字节转化成String
		System.out.println(new String(byteBuffer.array()));
		fileOutputStream.close();
	}

}

```

#### 3 . 使用一个 Buffer 完成文件的读取、写入


```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;

public class NIOFileChannel03 {

    public static void main(String[] args) throws Exception {

        FileInputStream fileInputStream = new FileInputStream("1.txt");
        FileChannel fileChannel01 = fileInputStream.getChannel();
        FileOutputStream fileOutputStream = new FileOutputStream("2.txt");
        FileChannel fileChannel02 = fileOutputStream.getChannel();

        ByteBuffer byteBuffer = ByteBuffer.allocate(512);
        
        while (true) { //循环读取

            //这里有一个重要的操作，一定不要忘了
            /*
            public final Buffer clear() {
                position = 0;
                limit = capacity;
                mark = -1;
                return this;
            }
            */
            byteBuffer.clear(); //清空 buffer
            int read = fileChannel01.read(byteBuffer);
            System.out.println("read = " + read);
            if (read == -1) { //表示读完
                break;
            }

            //将 buffer 中的数据写入到 fileChannel02--2.txt
            byteBuffer.flip();
            fileChannel02.write(byteBuffer);
        }

        //关闭相关的流
        fileInputStream.close();
        fileOutputStream.close();
    }
}
```

#### 4 . 拷贝文件transferFrom 方法


```java
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.nio.channels.FileChannel;

public class NIOFileChannel04 {

    public static void main(String[] args) throws Exception {

        //创建相关流
        FileInputStream fileInputStream = new FileInputStream("d:\\a.jpg");
        FileOutputStream fileOutputStream = new FileOutputStream("d:\\a2.jpg");
        
        //获取各个流对应的 FileChannel
        FileChannel sourceCh = fileInputStream.getChannel();
        FileChannel destCh = fileOutputStream.getChannel();

        //使用 transferForm 完成拷贝
        destCh.transferFrom(sourceCh, 0, sourceCh.size());

        //关闭相关通道和流
        sourceCh.close();
        destCh.close();
        fileInputStream.close();
        fileOutputStream.close();
    }
}
```

###  Buffer 和 Channel 注意事项
#### 1 . ByteBuffer 支持类型化的 put 和 get，put 放什么，get 取出什么，不然出现 BufferUnderflowException 异常


```java
import java.nio.ByteBuffer;

public class NIOByteBufferPutGet {

    public static void main(String[] args) {
        
        //创建一个 Buffer
        ByteBuffer buffer = ByteBuffer.allocate(64);

        //类型化方式放入数据
        buffer.putInt(100);
        buffer.putLong(9);
        buffer.putChar('尚');
        buffer.putShort((short) 4);

        //取出
        buffer.flip();
        
        System.out.println();
        
        System.out.println(buffer.getInt());
        System.out.println(buffer.getLong());
        System.out.println(buffer.getChar());
        System.out.println(buffer.getShort());
    }
}
```

#### 2 .  普通 Buffer 转成只读 Buffer


```java

import java.nio.ByteBuffer;

public class ReadOnlyBuffer {

    public static void main(String[] args) {

        //创建一个 buffer
        ByteBuffer buffer = ByteBuffer.allocate(64);

        for (int i = 0; i < 64; i++) {
            buffer.put((byte) i);
        }

        //读取
        buffer.flip();

        //得到一个只读的 Buffer
        ByteBuffer readOnlyBuffer = buffer.asReadOnlyBuffer();
        System.out.println(readOnlyBuffer.getClass());

        //读取
        while (readOnlyBuffer.hasRemaining()) {
            System.out.println(readOnlyBuffer.get());
        }

        readOnlyBuffer.put((byte) 100); //ReadOnlyBufferException
    }
}
```

#### 3 . NIO 中 MappedByteBuffer，可以让文件直接在堆外内存修改


```java

import java.io.RandomAccessFile;
import java.nio.MappedByteBuffer;
import java.nio.channels.FileChannel;

/**
 * 说明 1.MappedByteBuffer 可让文件直接在内存（堆外内存）修改,操作系统不需要拷贝一次
 */
public class MappedByteBufferTest {

    public static void main(String[] args) throws Exception {

        RandomAccessFile randomAccessFile = new RandomAccessFile("1.txt", "rw");
        //获取对应的通道
        FileChannel channel = randomAccessFile.getChannel();

        /**
         * 参数 1:FileChannel.MapMode.READ_WRITE 使用的读写模式
         * 参数 2：0：可以直接修改的起始位置
         * 参数 3:5: 是映射到内存的大小（不是索引位置），即将 1.txt 的多少个字节映射到内存
         * 可以直接修改的范围就是 0-5
         * 实际类型 DirectByteBuffer
         */
        MappedByteBuffer mappedByteBuffer = channel.map(FileChannel.MapMode.READ_WRITE, 0, 5);

        mappedByteBuffer.put(0, (byte) 'H');
        mappedByteBuffer.put(3, (byte) '9');
        mappedByteBuffer.put(5, (byte) 'Y');//IndexOutOfBoundsException

        randomAccessFile.close();
        System.out.println("修改成功~~");
    }
}
```

#### 4. NIO 还支持通过多个 Buffer（即 Buffer数组）完成读写操作，即 Scattering 和 Gathering


```java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Arrays;

/**
 * Scattering：将数据写入到 buffer 时，可以采用 buffer 数组，依次写入 [分散]
 * Gathering：从 buffer 读取数据时，可以采用 buffer 数组，依次读
 */
public class ScatteringAndGatheringTest {

    public static void main(String[] args) throws Exception {
        
        //使用 ServerSocketChannel 和 SocketChannel 网络
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        InetSocketAddress inetSocketAddress = new InetSocketAddress(7000);

        //绑定端口到 socket，并启动
        serverSocketChannel.socket().bind(inetSocketAddress);

        //创建 buffer 数组
        ByteBuffer[] byteBuffers = new ByteBuffer[2];
        byteBuffers[0] = ByteBuffer.allocate(5);
        byteBuffers[1] = ByteBuffer.allocate(3);

        //等客户端连接 (telnet)
        SocketChannel socketChannel = serverSocketChannel.accept();

        int messageLength = 8; //假定从客户端接收 8 个字节

        //循环的读取
        while (true) {
            int byteRead = 0;

            while (byteRead < messageLength) {
                long l = socketChannel.read(byteBuffers);
                byteRead += l; //累计读取的字节数
                System.out.println("byteRead = " + byteRead);
                //使用流打印,看看当前的这个 buffer 的 position 和 limit
                Arrays.asList(byteBuffers).stream().map(buffer -> "position = " + buffer.position() + ", limit = " + buffer.limit()).forEach(System.out::println);
            }

            //将所有的 buffer 进行 flip
            Arrays.asList(byteBuffers).forEach(buffer -> buffer.flip());
            //将数据读出显示到客户端
            long byteWirte = 0;
            while (byteWirte < messageLength) {
                long l = socketChannel.write(byteBuffers);//
                byteWirte += l;
            }
            
            //将所有的buffer进行clear
            Arrays.asList(byteBuffers).forEach(buffer -> {
                buffer.clear();
            });
            
            System.out.println("byteRead = " + byteRead + ", byteWrite = " + byteWirte + ", messagelength = " + messageLength);
        }
    }
}
```

## 5. Selector（选择器）

![[0b37a3b751ec9aa08efa75ace30e23c4_MD5.png|1050]]

1. Java 中的 NIO 可以用一个线程，处理多个客户端连接，就会使用到 Selector（选择器）
2. 多个 Channel 以事件的方式注册到 Selector
3. 只有在连接通道有真正的读写事件的时候，才会进行读写，减少系统开销
4. 避免了多线程之间的上下文切换导致的开销


```java
//Selector 类是一个抽象类，常用方法和说明如下：
public abstract class Selector implements Closeable{
	public static Selector open();//监控所有注册的通道，当其中有IO操作可以进行时，将SelectionKey加入到内部的集合中并返回，参数用来设置超时时间
	public Set<SelectionKey> selectedKey();//从内部集合中得到所有的SelectionKey	
}
 ```

以下是具体使用方法

##### 1.创建选择器


```java
Selector selector = Selector.open();
```
##### 2.将通道注册到选择器上

```java
ServerSocketChannel ssChannel = ServerSocketChannel.open();
ssChannel.configureBlocking(false);
ssChannel.register(selector,SelectionKey.OP_ACCEPT);
```

将通道注册到选择器上，还需要指定要注册的具体事件，主要有以下几类：
- SelectionKey. OP_CONNECT
- SelectionKey. OP_ACCEPT
- SelectionKey. OP_READ
- SelectionKey. OP_WRITE

他们在 SelectionKey 的定义如下：

```java
public static final int OP_READ = 1 << 0;
public static final int OP_WRITE = 1 << 2;
public static final int OP_CONNECT = 1 << 3;
public static final int OP_ACCEPT = 1 << 4;
```
##### 3. 监听事件

```java
int num = selector.select();
```

使用 `select()` 方法来监听到达的事件，它会一直阻塞知道有至少一件事件到达。

##### 4. 获取到达的事件

```java
Set<SelectionKey> keys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = keys.iterator();
while(keyIterator.hasNext()){
	SelectionKey keyu = keyIterator.next();
	if(key.isAcceptabnle()){
	// ...
	}else if(key.isReadable()){
	// ...
	}
	keyIterator.remove();
}
```
##### 5. 时间循环

因为一次 select() 调动不能处理完所有的事件，并且服务器端有可能需要一直监听事件，因此服务器端处理时间的代码一般会放在一个死循环内。

```java
while (true) { 
	int num = selector.select(); 
	Set<SelectionKey> keys = selector.selectedKeys(); 
	Iterator<SelectionKey> keyIterator = keys.iterator(); 
	while (keyIterator.hasNext()) { 
		SelectionKey key = keyIterator.next(); 
		if (key.isAcceptable()) { 
		  // ... 
		} else if (key.isReadable()) {
		  // ... 
		} 
		keyIterator.remove(); 
	}
}
```

套接字 NIO 实例

```java
public class NIOServer {
 	public static void main(String[] args) throws IOException {
		Selector selector = Selector.open();
		ServerSocketChannel ssChannel = ServerSocketChannel.open();
		ssChannel.configureBlocking(false);
		ssChannel.register(selector, SelectionKey.OP_ACCEPT);
		ServerSocket serverSocket = ssChannel.socket();
		InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8888);
		serverSocket.bind(address);
		while(true){
			selector.select();
			Set<SelectionKey> keys = selector.selectedKeys();
			Iterator<SelectionKey> keyIterator = keys.iterator();
			while (keyIterator.hasNext()){
				SelectionKey key = keyIterator.next();
				if (key.isAcceptable()) {
					ServerSocketChannel ssChannel1 = (ServerSocketChannel) key.channel();// 服务器会为每个新连接创建一个 SocketChannel SocketChannel sChannel = ssChannel1.accept();
					sChannel.configureBlocking(false);// 这个新连接主要用于从客户端读取数据 
					sChannel.register(selector, SelectionKey.OP_READ);
				}else if (key.isReadable()) {
					SocketChannel sChannel = (SocketChannel) key.channel();
					System.out.println(readDataFromSocketChannel(sChannel));
					sChannel.close();
				}
 			keyIterator.remove();
			}
		}
 	}
	private static String readDataFromSocketChannel(SocketChannel sChannel) throws IOException {
		ByteBuffer buffer = ByteBuffer.allocate(1024);
		StringBuilder data = new StringBuilder();
		while(true) {
			buffer.clear();
			int n = sChannel.read(buffer);
			if (n == -1) {
				break;
			}
			buffer.flip();
			int limit = buffer.limit();
			char[] dst = new char[limit];
			for (int i = 0;i < limit;i++) {
				dst[i] = (char) buffer.get(i);
			}
			data.append(dst);
			buffer.clear();
 		}
 		return data.toString();
 	}
 }
```

```java
public class NIOClient {
	public static void main(String[] args) throws IOException{
		Socket socket = new Socket("127.0.0.1"，8888);
		OutputStream out = socket.getOutputStream();
		String s = "hello world";
		out.write(s.getBytes());
		out.close();
	}
}
```
## 6. 典型的多路复用 IO 实现

目前流程的多路复用 IO 实现主要宝库了四种：select、poll、epoll、kqueue。以下是其特性及区别：

| IO 模型  | 相对性能 |       关键思路       |   操作系统    |                                             Java 支持情况                                              |
| :----: | :--: | :--------------: | :-------: | :------------------------------------------------------------------------------------------------: |
| select |  较高  |     Reactor      | Win/Linux | 支持，Reactor 模式（反应器设计模式）。Linux kernels 2.4 内核版本之前，默认用的是 select ；目前 windows 下对吧同步 IO 的支持，都是 select 模型 |
|  poll  |  较高  |     Reactor      |   Linux   |            Linux 下的 Java 的 NIO 框架，Linux kernels 2.6 内核版本之前使用 poll 进行支持。也是使用的 Reactor 模式            |
| epoll  |  高   | Reactor/Proactor |   Linux   |                               Linux kernels 2.6 内核版本之后使用 epoll 进行支持                                |
| kqueue |  高   |     Proactor     |   Linux   |                                           目前 Java 版本不支持                                            |

### 1. Reactor事件驱动模型

![[10d0080498aba2649f1c04067965b579_MD5.png|1025]]

从图上可知：一个完整的 Reactor 事件驱动模型是有四个部分组成：客户端 Client，Reactor，Acceptor 和时间处理 Handler。其中 Acceptor 会不间断的接收客户端的连接请求，然后通过 Reactor 分发到不同 Handler 进行处理。改进后的 Reactor 有如下优点：
- 虽然同是由一个线程接收连接请求进行网络读写，但是 Reactor 将客户端连接，网络读写，业务处理三大部分拆分，从而极大提高工作效率。
- Reactor 是以事件驱动的，相比传统 IO 阻塞式的，不必等待，大大提升了效率。

### 2. Reactor 模型——业务处理和 IO 分离

在上面的处理模型中，由于网络读写是在同一个线程里面。在高并发情况下，会出现两个瓶颈：
- 高频率的读写事件处理
- 大量的业务处理

基于上述瓶颈，可以将业务处理和 IO 读写分离出来：

![[37b1874d4aabb48cd6b511cba09fa70b_MD5.png|1025]](img/37b1874d4aabb48cd6b511cba09fa70b_MD5.png)


如图可以看出，相对基础 Reactor 模型，该模型有如下特点：
- 使用一个线程进行客户端连接和网络读写
- 客户端连接之后，将该连接交给线程池进行加解码以及业务处理

这种模型在接收请求进行网络读写的同时，也在进行业务处理，大大提高了系统的吞吐量。但是也有不足的地方：
- 网络读写是一个比较消耗 CPU 的操作，在高并发的情况下，将有大量的客户端需要网络读写，此时一个线程处理不了这么多的请求。

### 3. Reactor——并发读写

由于高并发的网络读写是系统一个瓶颈，所以针对这种情况，改进了模型，如图所示：

![[870a39b7312d1d18324d3aefa613ab37_MD5.png|1025]](img/870a39b7312d1d18324d3aefa613ab37_MD5.png)

由图可以看出，在原有 Reactor 模型上，同时将 Reactor 拆分成 mainReactor 和 subReactor 。其中 mainReactor 主要负责客户端的请求连接，subReactor 通过一个线程池进行支撑，主要负责网络读写，因为线程池的原因，可以进行多线程并发读写，大大提升了网络读写的效率。业务处理也是通过线程池进行。通过这种方式，可以进行百万级别的连接。

### 4. Reactor 模型示例

对于上述的 Reactor 模型，主要有三个核心需要实现：Acceptor，Reactor 和 Handler。具体实现代码如下：

```java
public class Reactor implements Runnable{
	private final Selector selector;
	private final ServerSocketChannel serverSocket;

	public Reactor(int port) throws IOException{
		serverSocket = ServerSocketChannel.open();//创建服务端的ServerSocketChannel
		serverSocket.configureBlocking(false);//设置为非阻塞模式
		selector = Selector.open();//创建一个selector选择器
		SelectionKey key = serverSocket.register(selector,SelectionKey.OP_ACCEPT);
		serverSocket.bind(new InetSocketAddress(port));//绑定服务端端口
		key.attach(new Acceptor(serverSocket));//为服务端Channel绑定一个Acceptor
	}
	@Override
	public void run(){
		try{
			while(！Thread.interrupted()){
				selector.select();//服务端使用一个线程不停接收连接请求
				Set<SelectionKey> keys = selector.selectedKeys();
				Iterator<SelectionKey> itetrator = keys.iterator();
				while(iterator.hasNext()){
					dispatch(iterator.next());
					iterator.remove();
					}
				selector.selectNow();
				}
		}catch(IOException e){
			e.printStackTrace();
		}
	}

	private void dispatch(SelectionKey key) throws IOException{
		//这里的attachment也即前面的为服务端Channel绑定的Acceptor，调用其run()方法进行分发
		Runnable attachment = (Runable)key.attachment();
		attachment.run();
	}

}

```

这里Reactor首先开启了一个ServerSocketChannel，然后将其绑定到指定的端口，并且注册到了一个多路复用器上。接着在一个线程中，其会在多路复用器上等待客户端连接。当有客户端连接到达后，Reactor就会将其派发给一个Acceptor，由该Acceptor专门进行客户端连接的获取。下面我们继续看一下Acceptor的代码：

```java
public class Acceptor implements Runnable{
	private final ExecuteorService executor = Exxcutors.newFixedThreadPool(20);

	private final ServerSocketChannel serverSocket;

	public Acceptor(ServerSocketChannel serverSocket){
		this.serverSocket = serverSocket;
		}

	@Override
	public void run(){
		try{
			SocketChannel channel = serverSocket.accept();
			if(null != channel){
				executor.execute(new Handler(channel));
			}
		}catch(IOException e){
			e.printStackTrace();
		}
	}
}
```

这里可以看到，在Acceptor获取到客户端连接之后，其就将其交由线程池进行网络读写了，而这里的主线程只是不断监听客户端连接事件。下面我们看看Handler的具体逻辑：

```java
public class Handler implements Runnable{
	private volatile static Selector selector;
	private final SocketChannel channel;
	private SelectionKey key;
	private volatile ByteBuffer input = ByteBuffer.allocate(1024);
	private volatile ByteBuffer output = ByteBuffer.allocate(1024);

	public Handle(SocketChannel channel) throws IOException{
		this.channel = channel;
		channel.configureBlocking(false);//设置客户端连接为非阻塞模式
		selector = Selector.open();//为客户端创建一个选择器
		key = channel.register(selector,SelectionKey.OP_READ);//注册客户端Channel的读事件	
	}

	@Override
	public void run(){
		try{
			while(selector.isOpen() && channel.isOpen()){
			 	Set<SelectionKey> keys = select();//等待客户端事件发生
			 	Iterator<SelectionKey> iterator = keys.iterator();
				while(iterator.hasNext()){
					SelectionKey key = iterator.next();
					iterator.remove();

					//如果当前是读事件，则读取数据
					if(key.isReadable()){
						read(key);
					}else if(key.isWritable()){
						write(key)
					}
				}
			}	
		}catch(IOException e){
		e.printStackTrace();
		}
	}
	//读取客户端发送的数据
	private void read(SelectionKey key) throws IOException{
		channel.read(input);
		if(input.position() == 0){
			return ;
		}
		input.flip();
		process();//对读数据进行业务处理
		input.clear();
		key.interstOps(SelectionKey.OP_WRITE);//读取完成后监听写入事件
	}
	private void write(SelectionKey key) throws IOException{
		output.flip();
		if(channel.isOpen()){
			channel.write(output);//当有写入事件时，将业务处理的结果写入到客户端Channel中
			key.channel();
			channel.close();
			output.clear();
		}
	}
	//进行业务处理，并且获取处理结果。本质上，基于Reactor模型，如果这里成为处理瓶颈，则将处理过程放入到线程池里面即可，并且使用一个Future获取处理结果，最后写入到客户端Channel中
	private void process(){
		byte[] bytes = new byte[input.remaining()];
		input.get(bytes);
		String message = new String(bytes,CharsetUtil.UTF_8);
		System.out.println("receive message from client: \n" +message);
		output.put("hello client".getBytes());
	}
}
```

在 Handler 中，主要进行的就是每个客户端 Channel 创建一个 Selector，并且监听该 Channel 的网络读写事件。当有事件到达时，进行数据的读写，而业务操作交友具体的业务线程池处理。

# 三、AIO（ Asynchronous I/O）

1. JDK7 引入了 Asynchronous I/O，即 AIO。在进行 I/O 编程中，常用到两种模式：Reactor 和 Proactor。Java 的 NIO 就是 Reactor，当有事件触发时，服务器端得到通知，进行相应的处理
2. AIO 即 NIO2.0，叫做异步不阻塞的 IO。AIO 引入异步通道的概念，采用了 Proactor 模式，简化了程序编写，有效的请求才启动线程，它的特点是先由操作系统完成后才通知服务端程序启动线程去处理，一般适用于连接数较多且连接时间较长的应用

### 1. 异步 IO

之前主要介绍了阻塞式同步 IO，非阻塞式同步 IO，多路复用 IO 这三种 IO 模型。而异步 IO 是采用“订阅-通知”模式，即应用程序想操作系统注册 IO 监听，然后继续做自己的实行。当操作系统发生 IO 时间，并且准备好数据后，主动通知应用程序，触发相应的函数：

![[3a01157436842562f7f21f8b4c4549d4_MD5.png|1025]](img/3a01157436842562f7f21f8b4c4549d4_MD5.png)

和同步 IO 一样，异步 IO 也是由操作系统进行支持的。Windows 系统提供了一种异步 IO 技术：IOCP（I/O Completion Port，I/O 完成端口）；
Linux 下由于没有这种异步 IO 技术，所以使用的是 epoll（上文介绍过的一种多路复用 IO 技术的实现）对异步 IO 进行模拟。

### 2. Java AIO 框架解析

![[2ff37cd0e1f5f734f8d6209396a08897_MD5.png|1025]](img/2ff37cd0e1f5f734f8d6209396a08897_MD5.png)
以上结构主要是说明 JAVA AIO 中类设计和操作系统的相关性。


参考资料：
> [BIO、NIO、AIO区别详解](https://zhuanlan.zhihu.com/p/520809188?utm_id=0)

