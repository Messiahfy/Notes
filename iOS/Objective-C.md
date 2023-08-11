## 内存管理
Objective-C使用引用计数管理内存，引用计数为0则释放对象内存。

1. MRC（手动引用计数）
OC的基类NSObject包含以下方法用于内存管理
* alloc/new/copy/mutableCopy：生成并持有对象（引用计数+1）
* retain：持有对象（引用计数+1）
* release：释放对象（引用计数-1）
* dealloc：废弃对象

2. ARC（自动引用计数）
编译器在适当的位置插入释放内存的代码

* __strong：强指针（默认），指针为nil或作用域结束就自动释放
* __weak：弱指针，不影响引用计数，没有强指针引用时，__weak指针引用变为nil。主要是解决循环引用等内存泄漏问题
* __unsafe_unretained：也不影响引用计数，但一个对象没有强指针引用时，__unsafe_unretained指针引用不会为nil，而是野指针
* __autoreleasing：

> MRC情况下，函数返回对象，要给调用方用，所以函数内不能释放，所以要么使用自动释放池，要么由调用方release（比如new、alloc开头的函数)

> ARC情况下，函数返回对象，由于编译器会检查方法名是否以alloc/new/copy/mutableCopy开头，如果不是则自动将返回值的对象注册到autoreleasepool。

[iOS里的内存管理](https://www.jianshu.com/p/c3344193ce02)

苹果给出了三种需要手动添加@autoreleasepool 的情況：
1. 程序不是基于UI框架，比如普通命令行
2. 如果你编写的循环中创建了大量的临时对象；你可以在循环内使用 @autoreleasepool 在下一次迭代之前处理这些对象。在循环中使用 @autoreleasepool 有助于减少应用程序的最大内存占用。
3. 如果你创建了其他线程，一旦线程开始执行，就必须创建自己的 @autoreleasepool，否则，你的应用程序将存在内存泄漏。

[Objective-C Automatic Reference Counting (ARC)](https://clang.llvm.org/docs/AutomaticReferenceCounting.html)

## Blocks

## 通信方式
Protocol
Block
NSNotification
KVC、KVO


## 多线程
pthread c语言
NSThread oc
GCD c
NSOperation oc

Runtime：https://www.ianisme.com/ios/2019.html
Runloop：https://blog.ibireme.com/2015/05/18/runloop/