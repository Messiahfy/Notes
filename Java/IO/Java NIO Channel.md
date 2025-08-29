&emsp;&emsp;Java NIO的`Channel`与输入输出流类似，但有一些区别：
* `Channel`是双向的，可以读也可以写，输入输出流是单向的
* 从`Channel`读数据到`Buffer`中，从`Buffer`写数据到`Channel`，不论整体是读还是写，都要使用`Buffer`这个中转站

*例子*：以`FileChannel`为例，读取文件内容并打印（省略try和关闭操作）：
```
FileInputStream fio = new FileInputStream("C:\\Users\\Administrator\\Desktop\\a.txt");
FileChannel fileChannel = fio.getChannel();
ByteBuffer byteBuffer = ByteBuffer.allocate(48);
while (fileChannel.read(byteBuffer) != -1) {
      byteBuffer.flip();
      while (byteBuffer.hasRemaining()) {
            System.out.print((char) byteBuffer.get());
      }
      byteBuffer.clear();
}
```
---------------------------------------
`Channel`以是否可以使用`Selector`来完成非阻塞多路IO分为：`FileChannel`和`其他Channel`
![Channel分类](https://upload-images.jianshu.io/upload_images/3468445-fefabf5c02be9bae.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### 1.FileChannel
总是阻塞式的
###### 1.1 FileChannel的创建
`Java 7`之前只能通过`输入输出流`或者`RandomAccessFile`的`getChannel()`方法创建`FileChannel`
`Java 7开始`添加了静态方法`open(Path, OpenOption)`来创建

###### 1.2 读写数据
1. 从`FileChannel`读数据，并写入`Buffer`
```
fileChannel.read(byteBuffer)
```
`read()`方法还有一些重载，其中`read(buffer，position)`从文件的`position`位置开始读

2. 写数据到`FileChannel`
```
fileChannel.write(buffer)
```
-------------------------------------------------------
`read()`和`write()`都有一个可以传入`Buffer`数组的重载方法，可用于将数据传输到多个`Buffer`中，如果是写入`Buffer`，则是写满`Buffer`数组的第一个`Buffer`后接着写第二个，依次类推；读取则是读完第一个就开始读第二个，依次类推。

使用结束后要调用`close()`方法关闭通道

-------------------------------------------------------------------------------------

###### Truncate(long size)    截断
将此通道的文件截断为给定大小。如果`size`小于文件的当前`size`，文件将被截断，丢弃后面的字节，如果等于则不改变，如果大于当前的`size`，那么文件的`size`被设为该`size`。

###### force(boolean metaData)`  强制冲刷更新
此方法把`FileChannel`的所有更新强制写入到磁盘存储中。因为出于性能原因，操作系统可能会缓存数据在内存中，所有必要时需要调用`force`方法确保数据写入磁盘。
`metaData`参数设置是否也强制更新文件的元数据（权限等）到磁盘中。

###### transfers 传输数据到其他Channel
&emsp;&emsp;`FileChannel`可以从实现`ReadableByteChannel`的子类（`DatagramChannel`、`FileChannel`、`Pipe.SourceChannel`、`SocketChannel`）中获得数据。
&emsp;&emsp;也可以传输数据到实现了`WritableByteChannel`的子类（`DatagramChannel`、`FileChannel`、`Pipe.SinkChannel`、`SocketChannel`）
1. `transferFrom(ReadableByteChannel src, long position, long count)`
从给定的可读`src`中传输数据到此`FileChannel`。可以避免先从一个`Channle`中循环读取再循环写入此`FileChannel`的流程。


2. `transferTo(long position, long count, WritableByteChannel target)`
从此`FileChannel`传输数据到可写`target`。可以避免先从此`FileChannel`中循环读取再循环写入一个`Channle`的流程。

###### map(FileChannel.MapMode mode, long position, long size)
将此`FileChannel`的文件的某个范围直接映射到内存（应该是虚拟内存）中，返回`MappedByteBuffer`。
缓冲区和它代表的映射直到对象垃圾回收才失效。
映射一旦建立，就不依赖于创建此映射的`FileChannel`，关闭通道对映射的有效性没有影响。
对于大多数操作系统，将较小的文件映射到内存来读写是昂贵的，一般只将较大的文件映射到内存中。

*内存映射文件的性能分析：*
从代码层面上看，从硬盘上将文件读入内存，都要经过文件系统进行数据拷贝，并且数据拷贝操作是由文件系统和硬件驱动实现的，理论上来说，拷贝数据的效率是一样的。
但是通过内存映射的方法访问硬盘上的文件，效率要比read和write系统调用高，这是为什么？

read()是系统调用，首先将文件从硬盘拷贝到内核空间的一个缓冲区，再将这些数据拷贝到用户空间，实际上进行了两次数据拷贝；
map()也是系统调用，但没有进行数据拷贝，当缺页中断发生时，直接将文件从硬盘拷贝到用户空间，只进行了一次数据拷贝。
所以，采用内存映射的读写效率要比传统的read/write性能高。文件较大时对比优势就比较明显。

#### 2.网络和管道相关Channel
这一系列通道的读写等方法和`FileChannel`差不多，最大不同的是：
抽象类`SelectableChannel`的子类支持使用`Selector`，以下是它的子类：
* `SocketChannel`
* `ServerSocketChannel`
* `DatagramChannel`
* `Pipe.SourceChannel`
* `Pipe.SinkChannel`

可以看出，TCP/UDP和管道的通道支持使用`Selector`，实现一个线程非阻塞多路复用IO。而`FileChannel`是不支持使用`Selelctor`的。

`SelectableChannel`的子类使用`configureBlocking(boolean)`可以设置是否阻塞。


#### 3.Channel的中断
`FileChannel`和`SelectableChannel`的子类都继承了`AbstractInterruptibleChannel`，表明它们的IO操作都可以被中断。

```
 boolean completed = false;
 try {
     begin();
     completed = ...;    // Perform blocking I/O operation
     return ...;         // Return result
 } finally {
     end(completed);
 }
```