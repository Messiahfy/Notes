#### 1.概述
&emsp;&emsp;`Buffer`在与`Channel`交互时使用，即从`Channel`读数据到`Buffer`，从`Buffer`写数据到`Channel`。
&emsp;&emsp;`Buffer`（缓存区）本质上是一个可以读写数据的内存块。此内存块包含在NIO Buffer对象中，该对象提供了一组方法，可以更轻松地使用内存块。

##### Buffer Capacity，Limit and Position
&emsp;&emsp;`Buffer`是特定基本类型的线性有限元素序列。 除了它的内容，`Buffer`的基本属性是它的`Capacity`（容量），`Limit`（限制）和`Position`（位置）：

1.`Capacity`：缓冲区的容量是它包含的元素数。 容量永远不为负数，永远不会改变。缓冲区满后，想继续写入数据就必须先读取或者清空数据。
2. `Limit`：缓冲区的限制是不应读取或写入的第一个元素的索引。缓冲区的限制永远不会为负，并且永远不会超过其容量。写入模式下，`Limit`为容量大小，即最多可以写满缓冲区；而将写入模式`flip()`（翻转）为读取模式，`Limit`被设为写入时的`position`，即最多读取到之前写入到的位置，`position`被设为0。
3. `Position`：缓冲区的位置是要读取或写入的下一个元素的索引，最大为容量-1。 缓冲区的位置永远不会为负，并且永远不会超过其限制。从缓冲区读取数据时，可以从给定位置开始读取。

#### 2.分配（创建）缓冲区
通过任意类型的`Buffer`的静态方法`allocate(int capacity)`分配一个新的`buffer`。
```
//分配一个容量为48字节的字节缓冲区
ByteBuffer byteBuffer = ByteBuffer.allocate(48);
//分配一个容量为1024字符的字符缓冲区
CharBuffer charBuffer = CharBuffer.allocate(1024);
```

#### 3.读写数据
> &emsp;&emsp;**和输入输出流不一样的是，例如FileInputStream创建后就可以开始读取数据，因为文件数据已经存在；而NIO中由于Channel读写数据都要使用Buffer作为中转站，所以使用Buffer必然是先要写入数据，而后才能读取数据。
&emsp;&emsp;比如从文件读取数据：要从Channel写入数据到Buffer，再从Buffer读取数据；写入数据到文件，则是先写入数据到Buffer，再用Channel从Buffer读取数据写到文件。**

&emsp;&emsp;使用`get`和`put`的各种重载方法来读写数据，不传索引的重载是相对位置，即以当前的`position`为读写位置；而可以传入索引的重载方法，就将索引位置作为读写位置，且不影响`position`。
###### 3.1 写入数据到Buffer
写入数据到`Buffer`有两种情况：
1. 从`Channel`写数据到`Buffer`（整体上来说是读数据，但是要先写到`Buffer`再读取）
2. 自己通过`Buffer`的`put()`方法写数据到`Buffer`（整体上来说是写数据，先写到`Buffer`，`Channel`再从`Buffer`读取）

例子1：从`FileChannel`读数据写入`Buffer`，返回读取的字节数量，到结束则返回-1
```
int bytesRead = fileChannel.read(byteBuffer);
```
例子2：自己写数据到`Buffer`
```
byteBuffer.put((byte) 127);
```
`put()`方法还有很多类型，具体查看API文档

###### 3.2 flip()
`flip()`方法把一个`Buffer`从写入模式切换（翻转）为读取模式。实质是把`limit`设为当前`position`，并把`position`置0，如果有`mark`则丢弃。

一个例子：先写两次数据到`buffer`，然后调用`flip()`改为读取模式，从`buffer`读取写到`channel`。
```
buf.put(magic);    // 前置 header
in.read(buf);      // 从channel读数据写到buf的剩余部分，现在buf的内容是header + data
buf.flip();        // Flip buffer
out.write(buf);    // 把 header + data 写到channel
```
> 这些`write`和`read`方法，都要注意从哪读或者写到哪，注意数据传输方向。

###### 3.3 从Buffer读取数据
从`Buffer`读取数据有两种情况：
1. 从`Buffer`读取数据写入`Channel`
2. 使用`Buffer`的`get`方法自行读取数据

例子1：从`Buffer`读取数据，写到`Channel`中
```
int bytesWritten = fileChannel.write(buf);
```
例子2：从`Buffer`读取数据，返回一个字节
```
byte aByte = buf.get();
```
`get()`方法还有很多类型，具体查看API文档



#### 4. 部分常用方法
* `mark()`和`reset()`：和输入输出流一样，设置标记，后面可以返回到标记位置

不变关系
*0 <= mark <= position <= limit <= capacity*

---------------------------------------------------------------------------------------------------
从`Buffer`读取数据完成后，一般要准备再次写入，这时可以调用`clear()`或者`compact()`方法。
* `clear()`：清空`Buffer`。实质并没有修改数据，只是把`position`置0，`limit`设为`capacity`，`mark`失效。

`clear()`方法适合读取完数据，或者对未读数据不再使用的情况。而如果`Buffer`中仍有部分未读数据，也就是`position`和`limit`之间的数据，并且想稍后读取它，那么此时应该使用`compact()`方法。<br/>
* `compact()`：`compact()`方法会把`position`到`limit`之间的数据（未读数据）复制到`Buffer`的开头， 也就是说，索引`p = position()`处的字节被复制到索引0，索引`p + 1`处的字节被复制到索引1，依此类推，直到索引`limit() -  1`处的字节被复制到索引`n = limit() -  1  -  p`。 然后将缓冲区的`position`设置为`n + 1`（也就是最后一个未读元素的下一个位置），并将其`limit`设置为其容量。 `mark`（如果已定义）将被丢弃。此时写入数据将从最后一个未读元素之后开始，不会覆盖之前的数据。
---------------------------------------------------------------------------------------------------

* `rewind()` ：保持`limit`不变并将`position`设置为0。可用来多次从头读取
* `order`和`order(ByteOrder)`：获得和设置字节的小端或小端顺序。
* `remaining()`和`hasRemaining()`：返回`position`到`limit`的差值；返回是否还有剩余值
* `wrap(byte[] array)`和`wrap(byte[] array, int offset, int length)`：将字节数组包装到`Buffer`。`Buffer`的修改将导致数组被修改，反之亦然。`position`为0，`limit`和`capacity`为array.length。
* `position()`和`position(int newPosition)`：返回当前位置；设置当前位置