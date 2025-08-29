## 1.概述
&emsp;&emsp;`Selector`是`SelectableChannel`（例如`SocketChannel`等子类）的多路转换器，它可以检查一个或多个`Channel`，并确定哪些通道可用于读写等操作。这样单个线程可以管理多个`Channel`，从而管理多个网络连接。

![非阻塞IO多路复用模型](https://upload-images.jianshu.io/upload_images/6959686-94023606c413da5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/504/format/webp)

## 2.Selector的创建
&emsp;&emsp;使用`Selelctor`类的静态方法`open()`可以创建一个`selector`，此方法会调用默认的`SelectorProvider`来创建该`selector`。也可以通过自定义`SelelctorProvider`调用`openSelelctor()`方法创建。`selector`保持打开状态直到调用`close()`方法。
```
Selector selector = Selector.open();
```
## 3.使用Selector注册Channel
要使`Channel`单线程多路复用，必须用`Selector`注册`Channel`。如下：
```
channel.configureBlocking(false);//设置channel非阻塞
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);
```
&emsp;&emsp;`Channel`使用`Selector`必须设为非阻塞，这意味着不能使用`FileChannel`注册`Selector`，因为`FileChannel`不能被设为非阻塞。

&emsp;&emsp;注意`register()`方法的第二个参数，这是一个`Interest set`（兴趣集合），表明通过`Selector`监听`Channel`的哪些事件，有四种事件：
1. `SelectionKey.OP_CONNECT`
2. `SelectionKey.OP_ACCEPT`
3. `SelectionKey.OP_READ`
4. `SelectionKey.OP_WRITE`

&emsp;&emsp;`connect`表示`Channel`连接一个Server成功；`accept`表示一个`ServerSocketChannel`接收一个连接成功；`read`表示一个`Channel`已经准备好数据读取；`write`表示一个`Channel`准备好写入数据。

&emsp;&emsp;可以对多个事件感兴趣：
```
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;    
```

## 4. SelectionKey's
&emsp;&emsp;调用`Channel`的`register()`方法后，会返回一个`SelectionKey`对象，此对象包含了一些信息，可以调用它的一些方法来获取。
###### The interest set  监听兴趣集
监听兴趣集合是使用`Selector`注册`Channel`时设置的监听类型参数，可以通过`SelectionKey`来读或写兴趣集合。
```
//读
int interestSet = selectionKey.interestOps();

boolean isInterestedInAccept  = interestSet & SelectionKey.OP_ACCEPT;
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE;
```
```
//写
selectionKey.interestOps(SelectionKey.OP_READ);
```
###### The ready set  就绪集合
就绪集合是`Channel`准备好的操作集合。
```
//访问就绪集合
int readySet = selectionKey.readyOps();

boolean isReadyAccept  = readySet & SelectionKey.OP_ACCEPT;
boolean isReadyConnect = readySet & SelectionKey.OP_CONNECT;
boolean isReadyRead    = readySet & SelectionKey.OP_READ;
boolean isReadyWrite   = readySet & SelectionKey.OP_WRITE;
```
或者用如下四种方法：
```
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
```
###### Channel和Selector
```
Channel  channel  = selectionKey.channel();

Selector selector = selectionKey.selector();
```
###### An attached object (optional)
&emsp;&emsp;您可以将对象附加到`SelectionKey`，这是识别给定`Channel`或将更多信息附加到`Channel`的便捷方式。 例如，您可以将正在使用的`Buffer`与`Channel`或包含更多聚合数据的对象相关联。 以下是如何附加对象：
```
selectionKey.attach(theObject);

Object attachedObj = selectionKey.attachment();
```
&emsp;&emsp;您还可以在`register()`方法中使用`Selector`注册`Channel`时附加对象：
```
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
```

## 5.通过Selector选择Channel


## 完整例子
```
Selector selector = Selector.open();

channel.configureBlocking(false);

SelectionKey key = channel.register(selector, SelectionKey.OP_READ);


while(true) {

  int readyChannels = selector.select();

  if(readyChannels == 0) continue;


  Set<SelectionKey> selectedKeys = selector.selectedKeys();

  Iterator<SelectionKey> keyIterator = selectedKeys.iterator();

  while(keyIterator.hasNext()) {

    SelectionKey key = keyIterator.next();

    if(key.isAcceptable()) {
        // a connection was accepted by a ServerSocketChannel.

    } else if (key.isConnectable()) {
        // a connection was established with a remote server.

    } else if (key.isReadable()) {
        // a channel is ready for reading

    } else if (key.isWritable()) {
        // a channel is ready for writing
    }

    keyIterator.remove();
  }
}
```