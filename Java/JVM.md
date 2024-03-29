## 类加载过程
读取Class文件，将它转换为对应的静态数据结构存储在方法区，并在堆中生成一个便于用于调用的java.lang.Class类型的对象

Class文件可以来源于网络、本地文件、二进制流。。。

1. 加载
2. 验证：文件格式、元数据、字节码验证、符号引用验证。
3. 准备：给静态变量赋默认值
4. 解析：符号引用转为直接引用。比如A和B两个Java文件，编译为两个单独的class文件，A引用B只能是一个字符串来表示，而到运行时，加载了A，发现B还没有加载，就会去加载B，并且把引用改为内存中的地址（静态解析）。不过，如果是抽象类或者接口，就要在运行时再去修改为内存地址（动态解析）。
5. 初始化：也就是对象的构造过程，先申请内存，再调用初始化函数，最后让变量指向该内存
6. gc

### 类加载器双亲委派
1. Bootstrap ClassLoader：加载jre/lib目录中的类。启动类加载器不能被Java程序直接引用，如果要让它加载类就使用null来表示它（比如String的）。
2. Extension ClassLoader：加载jre/lib/ext目录的类
3. Application ClassLoader：加载classpath中的类，`ClassLoader.getSystemClassLoader()`也称为系统加载器，默认的
4. 自定义类加载器：可以去加载自己指定位置的类

类加载器内部都有一个缓存，记录已加载的类 缓存在native，避免重复加载。

双亲委派的主要设计目的是为了安全，比如自己写了一个java.lang.Object类，双亲委派模型的流程会让Bootstrap ClassLoader加载java自带的。

双亲委派模型可以破坏：越基础的类由越上层的ClassLoader进行加载，但如果基础类需要加载非基础的类，那就需要上层的类加载器去调用下层的类加载器，比如Thread类的ContextClassLoader就是为了这个目的设计的，上层类加载器取出给Thread设置的ContextClassLoader，让这个ContextClassLoader去加载类。比如JDBC是一个操作数据库的标准，而不同数据库例如mysql需要提供对JDBC标准的实现，也就是driver包，JDBC相关接口在rt.jar中，会由Bootstrap ClassLoader加载，数据库driver则是我们自己放到classpath中，就需要Application ClassLoader或者我们自定义类加载器去加载。Thread的ContextClassLoader默认继承自创建该线程的线程，如果线程没有设置，就使用Application ClassLoader。

## 运行时内存结构
方法区是规范，jdk 8以前用永久代实现，存放被JVM加载的类信息、常量、静态变量等；jdk 8开始，用元空间（meta space）实现方法区，存放了类信息、运行时常量池等，而常量池和静态变量在堆中存放。


## Class的文件结构
类字节码中的常量池包含了链接表（对其他class文件的引用）

## GC
Java才用可达性算法（根搜索）判断对象是否需要回收。内存回收有标记-清除、复制、标记-整理等算法，因为不同对象有不同的生命周期，所以JVM将内存划为新生代和老生代，新生代采用复制算法，因为新生代每次回收后存活的对象是少数，所以复制并不会付出很多的代价，而老生代使用标记-整理之类的算法。

## 对象的内存布局
使用JOL库可以查看对象的内存布局

普通对象的内存布局：
1. 对象头：mark word（8个字节）、类型指针（在64位虚拟机上，是64位也就8个字节，但可能开启了指针压缩就是4个字节）
2. 实例数据
3. padding对齐：必须被8整除，所以要补齐到8的整数倍字节

> 可以用 java -XX:PrintCommandLineFlags -version，可以查看是否有UseCompressesClassPoints（开启类指针压缩）

数组的内存布局：
1. 对象头：mark word、类型指针
2. 数组长度：4个字节
3. 实例数据
4. padding对齐

使用new创建对象时，会先申请内存，再赋默认值，最后执行初始化代码（构造函数）赋初始值。例如双重检验单例就是使用volatile的有序性来避免这个顺序变化。

## 虚引用
强、软、弱三种引用都比较好理解它的使用场景，而虚引用似乎并没有什么用，通过它并不能拿到引用的对象，那么Java设计它的目的是什么呢？实际上使用它是必须配合ReferenceQueue，在对象被回收时就会在引用队列中取到引用，从而监听到对象被回收了。比如管理堆外内存，对 directByteBuffer 增加一个虚引用，当强引用没有的时候，它会马上被垃圾回收，通过referenceQueue就可以监听到，然后释放堆外内存。弱引用也可以用来做对象回收的监听，但是通过弱引用获取到对象赋值给一个变量，就可以对它增加强引用，而虚引用可以避免这个情况。（Reference内的processPendingReferences方法，判断了Cleaner类，directByteBuffer正是使用了Cleaner）