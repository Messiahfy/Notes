## 1. 概述
&emsp;&emsp;`Okio`是一个补充了`java.io`和`java.nio`的库，使访问、存储和处理数据变得更加容易。它最初是作为Android中包含的功能强大的HTTP客户端OkHttp的一个组件。

[OKio - 重新定义了“短小精悍”的IO框架](https://juejin.im/post/5856680c8e450a006c6474bd)

## 2. ByteStrings 和 Buffers
Okio围绕两种类型构建，它们将大量功能集成到一个简单的API中：

* `ByteString`是一个不可变的字节序列（数组）。对于字符数据，**String**（`Java`的`String`类是字符序列）是基础。`ByteString`可以很容易的将二进制数据转化为一个值：编码和解码为16进制、base64、UTF-8。
* `Buffer`是一个可变的字节序列（段的链表）。不用预先设置缓存区大小，将缓存区作为队列读取和写入：写入数据到末尾并从前面读取，不用管理position、limits或者capacities。

在内部，`ByteString`和`Buffer`做了一些聪明的事情来节省CPU和内存。如果将UTF-8字符串编码为`ByteString`，它会缓存对该字符串的引用，这样如果以后对其进行解码，则无需执行任何操作。

`Buffer`实现为段（`segment`）的链表。将数据从一个`Buffer`移动到另一个`Buffer`时，它会重新分配段的所有权，而不是复制数据。这种方法对多线程程序特别有用：与网络通信的线程可以与工作线程交换数据而无需任何复制或仪式。

## 3.Sources 和 Sinks
&emsp;&ensp;`java.io`设计的一个优雅部分是如何转换分层流例如加密和压缩等。 `Okio`包含自己的流类型，称为`Source`和`Sink`，其工作方式类似于`InputStream`和`OutputStream`，但有一些主要区别：
* **超时**：`Source`和`Sink`都提供了底层IO机制的超时访问。与`java.io socket`流不同，`read()`和`write()`都调用了honor超时（暂不了解）。
* **易于实现**： `Source`接口声明了三个方法：`read()`、`close()`和`timeout()`。 没有像`available`或单字节读取这样的风险导致正确性和性能意外。
* **易于使用**：尽管`Source`和`Sink`的实现只有三个方法去编写，但是调用者可以使用`BufferedSource`和`BufferedSink`接口获得丰富的`API`。这些接口提供了所有的需求。
* **字节流和字符串流之间没有人为的区别**：所有的数据，读写它作为字节、UTF-8字符串、大端32位整型、小端short类型，任何你想要得。没有多余的`InputStreamReader`。
* **易于测试**：`Buffer`类实现了`BufferedSource`和`BufferedSink`，因此您的测试代码简单明了。

`Source`和`Sink`与`InputStream`和`OutputStream`交互操作。 您可以将任何`Source`视为`InputStream`，并且可以将任何`InputStream`视为`Source`。 同样适用于`Sink`和`OutputStream`。
> `Source`和`Sink`实质就是`InputStream`和`OutputStream`的包装。
## 4.Okio的设计与使用
![Okio结构](https://upload-images.jianshu.io/upload_images/3468445-4bf01d11b5b4150c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
使用时一般会调用`Okio.buffer()`方法，会返回`RealBufferedSource`或`RealBufferedSink`实例，而这两个类的主要功能都是由内部的`Buffer`和`Source`或`Sink`配合来实现。  

使用Okio来读取数据的例子：
```
BufferedSource source = Okio.buffer(Okio.source(new File("C:\\Users\\huang\\Desktop\\a.txt")));
System.out.println(source.readByteString().string(Charset.forName("utf-8")));
```

`Buffer`实现了输入输出的核心工作，数据存储使用了`Segment`链表，`Segment`内部是一个字节数组，与`SegmentPool`共同实现缓存机制。
![Okio缓存模块](https://upload-images.jianshu.io/upload_images/3468445-83f3c8e553c06261.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


和NIO类似，Okio内部的读写也是**先写读到Buffer缓存区，再从缓存区读写**，Buffer既是Source也是Sink，所以读写都可以用这个Buffer作为中转。有缓存可以防止频繁GC