### 通过ServiceManager获取系统服务的跨进程通信情况
1. Linux内核初始化Binder驱动
2. 启动ServiceManager，打开Binder驱动，mmap将Binder中申请的内核内存映射到ServiceManager。然后执行binder_loop，先通过ioctl系统调用把BC_ENTER_LOOPER指令发送到Binder驱动，就会调用到binder驱动的binder_ioctl函数，告知Binder驱动后，ServiceManager开始循环执行任务，执行到`binder_thread_read`的`wait_event_freezable_exclusive`，等待从Binder驱动中读取数据，进入睡眠，等待客户端写入数据后唤醒
3. 应用程序进程启动时构造ProcessState对象，打开Binder驱动，mmap映射内存，然后创建IPCThreadState，循环执行getAndExecuteCommand()
4. 上述各部分准备好后，从应用程序使用AMS为例开始描述。阅读原理3部分，可知应用程序最终是拿到SM的代理，而代理中就持有一个SM的IBinder，它在Native层的实现就是`BpBinder`（BpBinder.cpp），由`ProcessState.getContextObject`创建，创建的BpBinder中会包含SM对应的handle标识0。然后执行`BpBinder.transact`-->`IPCThreadState.transact`-->`waitForResponse`-->`talkWithDriver`-->`Binder驱动的binder_ioctl`-->`binder_ioctl_write_read`-->`binder_thread_write`-->`binder_transaction`
5. 执行到Binder驱动的`binder_transaction`函数，通过handle标识找到目标进程对应的Binder_Node，SM启动的时候的已经预先创建好它的Binder_Node，经过一系列数据处理，把`tcomplete（binder_work类型）`放到发起方的待办队列，把`t（binder_transaction类型）`放到SM的待办队列中，然后唤醒SM。并且自身返回到`binder_ioctl_write_read`中，因为talkWithDriver中已经设置bwr.read_size大于0，所以会继续执行`binder_thread_read`
6. 执行到Binder驱动的`binder_thread_read`函数，虽然会执行wait_event_freezable，但是因为之前已经把`tcomplete`放到了发起方待办队列中，所以执行`binder_has_thread_work`会返回true，也就实际不会阻塞当前线程。由于之前执行`binder_transaction`的时候添加`tcomplete`时，已经设置了`BINDER_WORK_TRANSACTION_COMPLETE `，所以这里会设置`BR_TRANSACTION_COMPLETE`写到用户空间，然后本次`binder_ioctl`执行完毕。返回到`IPCThreadState.talkWithDriver`
7. 应用程序从`talkWithDriver`的`ioctl`之后继续执行，然后返回到`waitForResponse`中，此时命令为`BR_TRANSACTION_COMPLETE`，则正常情况再次循环，再次执行`talkWithDriver`到Binder驱动中（也就是同步的情况），如果时one way，就结束操作。如果再次执行binder_ioctl，此时bwr.write_size等于0，而bwr.read_size大于0，于是再次进入到`binder_thread_read`函数。此时thread->transaction_stack仍然不为NULL，但thread->todo队列已经为空，因为前面已经处理过thread->todo队列的内容了，然后就会执行`wait_event_freezable`休眠。现在应用进程休眠，等待被唤醒。
8. 回到SM，它被应用进程唤醒后，继续执行`binder_thread_read`的后续流程。由于用户进程在放`t（binder_transaction类型）`到SM的待办队列时，设置的是`BINDER_WORK_TRANSACTION`，所以会去取出`t`，获取相关数据，设置`BR_TRANSACTION`，写回用户空间。然后返回到`binder_ioctl`，将结果写回用户空间。再返回到`binder_loop`，执行`binder_parse`来处理从Binder驱动中读到的数据。
9. 执行`binder_parse`，由于`getService`传入的code为GET_SERVICE_TRANSACTION，对应SVC_MGR_GET_SERVICE，在svclist列表中查找对应名称的svcinfo，返回它的指针。封装为一个类型为`BINDER_TYPE_HANDLE`的`binder_object`，数据回复命令为`BC_REPLY`，通过`binder_send_reply`再次执行到`ioctl`，并且只有写操作。
10. 再次执行到`binder_thread_write`，命令为`BC_REPLY`，再次执行`binder_transaction`，本次将执行`reply`为true的情况，得到发起方进程和线程记录，然后执行到`BINDER_TYPE_HANDLE`分支，根据handle取出Binder驱动中的`binder_node`，也就是注册的服务的信息，然后设置`tcomplete`和`t`，唤醒应用进程。
11. 应用进程被唤醒后，在`binder_thread_read`中继续执行到`BINDER_WORK_TRANSACTION`分支，操作数据，然后返回到`binder_ioctl_write_read`中，执行后续的`copy_to_user`将数据传到用户空间。然后返回到`talkWithDriver`再返回到`waitForResponse`执行`BR_REPLY`分支，执行最开始执行`transact`传入的reply（Parcel类型）的`ipcSetDataReference`函数，然后一直返回到`ServiceManagerProxy`的`getService`方法，会使用得到的handle创建对应的`BpBinder`，最后返回到Java层。
12. 应用进程拿到了AMS对应handle的IBinder，就可以用它再次执行Binder通信流程与AMS通信。

### 注册和查找服务
1. 执行`ServiceManagerProxy.getservice`，会调用`Parcel.writeStrongBinder`把服务（IBinder）转换为特定数据结果放到Parcel中
2. Java中的`Parcel.writeStrongBinder`会调用C++的`Parcel.writeStrongBinder`
3. 调用C++的`Parcel.flatten_binder`，此时是处理本地binder，不会设置handle，但会设置cookie为该binder的指针
4. 执行到Binder驱动的binder_transaction方法的BINDER_TYPE_BINDER分支，添加binder_node到Binder驱动内当前进程对应的proc的binder_node红黑树内，然后binder_get_ref_for_node方法会设置handle值（递增的int值）
5. 唤醒ServiceManager，svcmgr_handler方法的SVC_MGR_ADD_SERVICE分支代码会添加该服务，取出Binder驱动中设置的handle值，将服务存起来。
6. 通过字符串名称以及handle为0向ServiceManager请求服务，ServiceManager根据字符串名称找到该服务，然后得到该服务的handle，调用Binder驱动
7. ServiceManager执行到Binder驱动的binder_transaction方法的BINDER_TYPE_HANDLE分支，把handle等数据放到了数据结构中，并且会通过binder_get_ref_for_node方法创建一个binder_ref（会记录它对应的目标进程的binder_node）放到Binder驱动中Client进程的binder_ref红黑树中
8. 唤醒客户端，取出数据写入Parcel，在用户空间读取，可以看到ServiceManagerProxy.getService中调用`Parcel.readStrongBinder`，调用unflatten_binder，得到handle并传给ProcessState的getStrongProxyForHandle方法，这里会创建BpBinder（包含了handle）并返回
9. 通过包含了handle的BpBinder执行到Binder驱动，就可以通过handle去自己进程对应的binder_proc里面找到binder_ref，通过binder_ref的target_node和target_proc唤醒该目标进程去跨进程执行任务。


### 通过bindService的跨进程通信情况
1. 使用bindService的方式跨进程通信，没有通过ServiceManager，而是通过AMS（当然，通过SM才能找到AMS），经过AMS的作用在于可以协调回调service进程的`onBind`等函数、以及client进程的`onServiceConnection`函数。服务端返回的IBinder一般是AIDL生产的类的实例，继承自Binder，Binder的构造函数中会创建对应的native层对象。通过Binder驱动把该IBinder对象传给客户端，先通过Parcel的`flatten_binder`把IBinder转换为`flat_binder_object`类型，会包含此Binder对象的指针，因为现在是把本进程的IBinder写入，会设置BINDER_TYPE_BINDER。
2. Binder驱动中，`binder_transaction`方法的BINDER_TYPE_BINDER分支，binder_get_ref_for_node会为这个IBinder对应的binder_ref类型生成desc值，也就会成flat_binder_object的handle值。
3. 然后这个flat_binder_object再传到客户端，并还原成IBinder，就已经包含了handle值，所以客户端用它发起跨进程调用，是可以在Binder驱动中找到对应的进程的，并且通过指针会找到具体的那个远程Binder对象，去执行BBinder.onTrascat。

Client：Proxy--BinderProxy--BpBinder
Server：Stub--Binder--BBinder

一次复制：如果有返回值的binder通信，当然还是会有两次复制

zygote：用socket，因为zygote自身用于fork其他进程，自身更简单更方便，如果自身用binder，fork也会复制binder线程


### 异常处理
在 Parcel.java 中，可以看到会把支持的各种异常对应特定的code来传递，如果不是支持的异常，会在createException方法中再抛出，寻找到`frameworks/base/core/jni/android_util_Binder.cpp`中`JavaBBinder`类（BBinder子类，Naive的服务类）的`onTransact`函数中catch不支持的异常则会打印出来。对客户端没有影响，只是不响应任何结果和异常