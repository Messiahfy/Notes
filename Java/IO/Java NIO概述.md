[一篇Java NIO的文章](http://tutorials.jenkov.com/java-nio/overview.html)

&emsp;&emsp;Java NIO有三个核心概念`Channel`、`Buffer`和`Selector`，其余的例如Pipe和FileLock仅仅是与三个核心组件配合使用的类。
&emsp;&emsp;NIO中的输入输出使用`Channel`，`Channel`有一点（只是有一点）像老IO中的输入输出流。从`Channel`中可以读数据到`Buffer`，从`Buffer`可以写数据到`Channel`，如下图：
![Java NIO: Channels 读数据到 Buffers, Buffers 写数据到 Channels
](https://upload-images.jianshu.io/upload_images/3468445-ea53af1c0ea49f77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### Channel的主要实现如下：
* FileChannel
* DatagramChannel
* SocketChannel
* ServerSocketChannel

**分别用于文件 IO、UDP和TCP的网络 IO**
关于`Channel`还有一些有趣的接口，在后续详细介绍
 
#### Buffer的主要实现如下：
* ByteBuffer
* CharBuffer
* DoubleBuffer
* FloatBuffer
* IntBuffer
* LongBuffer
* ShortBuffer

包含了可以通过IO发送的基本数据类型
还有一个**MappedByteBuffer**和内存映射文件一起使用，后面再具体分析。

#### Selector
&emsp;&emsp;`Selector`允许单个线程处理多个`Channel`。如果您的应用程序打开了许多连接（`Channel`），但每个连接只有较低的流量，这很方便。 例如，在聊天服务器中。
![一个线程使用 Selector 来处理3个 Channel](https://upload-images.jianshu.io/upload_images/3468445-28eaebc8feaa8453.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


&emsp;&emsp;使用`Selector`，要注册`Channel`。 然后调用它的`select()`方法， 此方法将阻塞，直到有一个已注册`Channel`的事件准备就绪。 一旦该方法返回，该线程就可以处理这些事件。 事件：比如传入连接，接收数据等。