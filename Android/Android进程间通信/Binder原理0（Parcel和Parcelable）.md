## Parcel
进程间通信传输的数据，本质上是复制一份一样的数据，Android中使用Parcel来包装需要传输的数据。对于基本类型，可以直接设置一样的值传输；对于引用类型，则需要实现Parcelable接口的相关方法，并且引用类型中的属性如果是引用类型，如果需要跨进程传输，则也需要实现Parcelable接口。
```
Parcel p = Parcel.obtain();
p.writeInt(1);
p.writeInt(2);
```
//读取前必须设置位置，否则写了两个int即8个字节，那么当前位置是9，则会从第9位置开始读，所以要先设为0
p.setDataPosition(0);

Log.e("测试测试", "onStart: " + p.readInt());
Log.e("测试测试", "onStart: " + p.readInt());
p.recycle();

#### Parcelable
实现Parcelable接口的对象可以通过Parcel来存取，比如常用的bundle就实现了Parcelable。
```
parcel.writeParcelable(Parcelable, int)  将此Parcelable的类名和数据写入Parcel，写入数据实际是调用Parcelable的writeToParcel方法
parcel.readParcelble(ClassLoader)  读取并返回一个新的Parcelable对象，实际调用Parcelable.Creator的createFromParcel(Parcel)方法
...
```

#### Bundle
Bundle实现了Parcelable，是一种特殊的type-safa的容器，采用键值对的方式存储数据。

#### Active Object
Parcel可以读写Active Object。通常存入Parcel的是对象的内容，而Active Object写入的则是它们的特殊标志引用。对于Active Object，从Parcel读取并不会重新创建一个内容一样的对象，而是原来那个被写入的实例。主要有两种类：
1. Binder 这里的Binder指的是Binder类的实例，通过Parcel将Binder对象写入，读取时可以获取到原始的Binder对象，或者它的特殊代理
2. FileDescriptor 文件描述符，因为传递后的文件描述符仍然基于相同的文件流操作，所以可以认为是Active Object的一种。

Java层的Parcel只是一个中介，实际的读写操作都在C++部分

## 匿名共享内存
Android提供的Java层共享内存API：MemoryFile、SharedMemory，内部还是使用Linux的mmap