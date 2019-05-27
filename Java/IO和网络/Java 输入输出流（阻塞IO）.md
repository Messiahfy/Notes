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