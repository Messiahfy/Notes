## 1.概述
输入输出流不会保存数据，只相当于一个数据通道。不论是读取文件还是读取http请求的响应流，都是读多少传多少，不是先全部放到输入流中才读取。直到关闭流，才断开连接（HTTP就是断开TCP的socket长连接）。HTTP响应头会直接拿到，而body则需要维持长链接传输，用okhttp的事件监听可以得出前面的结论。与C语言文件指针类似，输入输出流也会保存资源的位置和当前读取或写入到的位置等信息（信息估计在调用虚拟机的C++层次中）


![IO体系](https://upload-images.jianshu.io/upload_images/3468445-63f5122f674ee9ef.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
非流式：File、RandomAccessFile、FileDescriptor

其他：文件读取部分与安全相关的类，如：SerializablePermission类


![IO流体系](https://upload-images.jianshu.io/upload_images/3468445-602757ffde848cba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.IO流分类
1. `Memory 内存`
* 从/向内存数组读写数据: CharArrayReader、 CharArrayWriter、ByteArrayInputStream、ByteArrayOutputStream
* 从/向内存字符串读写数据 StringReader、StringWriter、StringBufferInputStream
```
     //文件转化为字节数组
    public static byte[] file2ByteArray(File file) {
        byte[] result = new byte[0];
        try (
                InputStream in = new FileInputStream(file);
                ByteArrayOutputStream out = new ByteArrayOutputStream()
        ) {
            byte[] buffer = new byte[1024 * 4];
            int len;
            while ((len = in.read(buffer)) != -1) {
                out.write(buffer, 0, len);
            }
            result = out.toByteArray();
        } catch ......
        return result;
    }

```

2. `Pipe 管道`  实现管道的输入和输出（进程间通信）: PipedReader、PipedWriter、PipedInputStream、PipedOutputStream

3. `File 文件流`。对文件进行读、写操作 ：FileReader、FileWriter、FileInputStream、FileOutputStream

4. `ObjectSerialization 对象输入、输出 `：ObjectInputStream、ObjectOutputStream

5. `DataConversion 数据流` 按基本数据类型读、写（处理的数据是Java的基本类型（如布尔型，字节，整数和浮点数））：DataInputStream、DataOutputStream

6. `Printing 包含方便的打印方法` ：PrintWriter、PrintStream

7. `Buffering缓冲`  在读入或写出时，对数据进行缓存，以减少I/O的次数：BufferedReader、BufferedWriter、BufferedInputStream、BufferedOutputStream

8. `Filtering 滤流`，在数据进行读或写时进行过滤：FilterReader、FilterWriter、FilterInputStream、FilterOutputStream过滤

9. `Concatenation`合并输入 把多个输入流连接成一个输入流 ：SequenceInputStream 

10. `Peeking Ahead` 通过缓存机制，进行预读 ：PushbackReader、PushbackInputStream

11. `Converting between Bytes and Characters` 按照一定的编码/解码标准将字节流转换为字符流，或进行反向转换（Stream到Reader,Writer的转换类）：InputStreamReader、OutputStreamWriter

## 3.部分常用类的解释
#### 字节流
###### 缓冲流
`BuffererdInputStream` 用缓冲流来包装其他输入流，可以每次存满缓冲区才读取，比默认每次read都要请求操作系统分发一个字节更加高效。

###### 预览回推
`PushBackInputStream` 读取一个值，如果并非期望的可以推回流中

###### 读入数值
`DataInputStream` 例如`readDouble`方法，将读入8个字节，返回一个`double`值
> 实现了`DataInput`接口，`DataOutput`同理

#### 字符文本流常用类
`OutputSteamWriter`

`PrintWriter`

和各种`Reader`


> 本质上一切都是字节流，没有字符流。字符只是根据编码集对字节流翻译之后的产物。Writer和Reader的作用只是可以帮我们读些字节流的时候，通过设置的编码集，转换字符和字节。比如同样的字节数据，根据不同的编码集，会转换为不同的字符。

------------------------------------------

这里列几个读入文本输入的非直接用输入流的`Java 7`开始的`Files`和`Path`方式
1.对于短小的文本文件
```
String content = new String(Files.readAllBytes(path), charset);
```
2.如果要一行行读入：
```
List<String> lines = Files.readAllLines(path, charset);
```
3.如果文件较大，可以使用Java 8的流
```
try(Stream<String> lines = Files.lines(path, charset)){
     ...
}
```

## 4. 文件原子修改
### 多线程
测试同一进程内Java多线程写入同一个File的同一个FileOutputStream：
```
// kotlin代码

fun main() {
    val file = File(".../filetest")
    val os = file.outputStream()
    val thread1 = thread {
        val s = "aaaaa\n"
        for (i in 1..10000) {
            os.write(s.toByteArray())
        }
    }
    val thread2 = thread {
        val s = "bbbbb\n"
        for (i in 1..10000) {
            os.write(s.toByteArray())
        }
    }
    val thread3 = thread {
        val s = "ccccc\n"
        for (i in 1..10000) {
            os.write(s.toByteArray())
        }
    }
    thread1.join()
    thread2.join()
    thread3.join()
    os.close()
}
```
* 结果总行数为3万行正确，每种打印都是1万行，顺序随机，但都是aaaaa、bbbbb、ccccc的情况，不会有aabbcc这类错误情况。如果把打印从5个字符扩大到1000个字符，测试结果也是一样。
* 如果同一文件的多个FileOutputStream实例，每种打印行数不一定是10000，总行数也会错误，顺序随机，仍保证不会有aabbcc这类错误情况。
* 但如果同一文件的多个FileOutputStream实例，使用append模块，则能保证总行数为3万行正确，每种打印都是1万行

> FileOutputStream默认为覆盖写入模式（O_TRUNC），可以设置append。

Java的文件读写基于Linux的write/read等系统调用，write调用可以保证同一fd下的原子性（老的linux版本可能有问题）。所以不会产生aabbcc的情况，但由于一般都是循环调用write，所以有顺序随机问题是正常的。如果多个fd则无法保证。

https://github.com/torvalds/linux/commit/9c225f2655e36a470c4f58dbbc99244c5fc7f2d4 linux 3.14修复write原子性bug

https://blog.csdn.net/dog250/article/details/78879600


c库的fwrite调用write时加了锁，所以即使在linux有bug的版本也正常

write调用能保证的是，不管它实际写入了多少数据，比如写入了n字节数据，在写入这n字节数据的时候，在所有共享文件描述符的线程或者进程之间，每一个write调用是原子的，不可打断的。举一个例子，比如线程1写入了3个字符’a’，线程2写入了3个字符’b’，结果一定是‘aaabbb’或者是‘bbbaaa’，不可能是类似‘abaabb’这类交错的情况。

如果两个进程没有共享同一个文件描述符，这种情况下没有任何保证

多线程共享同一个文件描述符号不会发生写入覆盖，文件长度符合预期
父子进程指向同一个文件表不会发生写入覆盖，文件长度符合预期
多进程/多线程打开同一份文件（不同文件描述符），不会发生内容交错，但是会发生内容覆盖（导致长度不符合预期），实际内容来自其中一个线程


write() 三部曲
1. 从文件表中获取偏移量
2. 从偏移量处开始写，更新文件长度
3. 更新文件表偏移量

对于 多线程共享同一个文件描述符 和 父子进程指向同一个文件表 的情况，这3个步骤不会被打断(没有出现数据覆盖或交叉的情况)
之所以会这样，原因在于文件表有一个读写锁，mutex_lock(&file->f_pos_lock) 会锁住文件表的 f_pos_lock 锁，其它线程试图调用 write() 写入数据时，只要 fd 指向的是同一个文件表，那么就必须等待锁释放

对于 多进程/多线程打开同一份文件（不同文件描述符） 的情况，这3个步骤中，步骤1和2之间可能被打断，步骤2和3之间也可能被打断，但是步骤2单独不会被打断(没有出现数据交叉的情况)。之所以会这样，原因在于 inode 索引节点也有一个读写锁 inode_lock(inode) 会锁住 inode，其它线程试图调用 write() 写入数据时，只要 fd 指向的是同一个 inode，那么就必须等待锁释放。但是因为不同进程/线程从文件表获取偏移量是独立并行执行的，向同一个偏移量开始写入数据，所以写入会发生覆盖

在 open() 的时候，加上 O_APPEND 就能解决 多进程/多线程打开同一份文件 不同描述符 数据覆盖的问题，原因是，在第二步锁住 inode 后，强制更新偏移量为当前文件的实际大小，第一步获取的偏移量将无效。

write使用O_APPEND才会行数正常

但数据行数可能和预期不同，多线程写文件不是原子的，因为写入时获取offset和文件写入是两个步骤，会导致数据相互覆盖，可以加锁解决，

### 事务性
但这里并非讨论多线程场景，而是修改文件要么完全写入，要么未写入，不能存在因为异常中断导致只写入了部分的情况.

操作系统的文件系统不支持atomic修改文件，同一文件系统内重命名文件（rename、move）一般是原子的（具体系统需要具体查阅资料确认）。

* 文件为 test.txt
* 创建 test.txt.tmp，并写入数据
* 将 test.txt 重命名为 test.txt.bak
* 将 test.txt.tmp 重命名为 test.txt
* 新的 test.txt 正确放置后，删除 test.txt.bak

如果新的 test.txt 正确放置前出现问题，则使用 test.txt.bak 来恢复。并且可能要配合flush、sync调用。

android.util.AtomicFile


如果一边写，一边读，FileInputStream read是获取实时的数据，比如读到index为5是最后一个数据，如果又写入了数据，那么可以继续读取index为6的数据。


缓存IO
* 写入：用户进程内的内存缓存 -- 内核缓冲区 -- 硬盘
将数据从用户空间复制到内核空间的缓存中。这时对用户程序来说写操作就已经完成，至于什么时候再写到磁盘中由作系统决定，除非显示地调用了sync同步命令。

* 读取：硬盘 -- 内核缓冲区 -- 用户进程内的内存缓存

进程在执行 write （使用缓冲 IO）系统调用的时候，实际上是将文件数据写到了内核的 page cache，它是文件系统中用于缓存文件数据的缓冲，所以即使进程崩溃了，文件数据还是保留在内核的 page cache，我们读数据的时候，也是从内核的 page cache 读取，因此还是依然读的是进程崩溃前写入的数据。（保证flush到内核）内核会找个合适的时机，将 page cache 中的数据持久化到磁盘。但是如果 page cache 里的文件数据，在持久化到磁盘化到磁盘之前，系统发生了崩溃，那这部分数据就会丢失了。

当然， 我们也可以在程序里调用 fsync 函数，在写完文件的时候，立刻将文件数据持久化到磁盘，这样就可以解决系统崩溃导致的文件数据丢失的问题。

#### AtomicFile
* 写文件：直接写new文件，成功就把new重命名为current，失败就删除new。中断的话，下次写入会覆盖写新的。
* 读文件：new和current都存在（中断的情况），就先删除new，读current；new不存在就直接读current

#### 先不考虑使用中的情况（AtomicDirectory）：
* 写目录：
    1. 如果current不存在，直接写current目录，如果成功结束就sync，如果失败，啥也不干，因为没有current所以没有备份，也就不需要恢复
    2. 如果current存在，先备份current为current_bak，然后写入到current目录，如果成功结束：则把current_bak删除，留下current；如果失败：则恢复，即删除current，把current_bak重命名为current目录
* 读目录：如果current_bak存在，说明还没有结束写入操作或者写入操作被中断，恢复备份：则删除current，然后把current_bak重命名为current目录，再读取current目录；如果current_bak不存在，则直接读current目录

只要备份文件存在，原本目录就被认为是无效的。调用者需要确保不能并发，并且只能通过此类来操作这里的目录。