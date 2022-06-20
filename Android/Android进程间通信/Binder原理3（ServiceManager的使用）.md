一个Binder Client如何使用ServiceManager，关键步骤仍然是熟悉的：
* 打开Binder驱动
* 执行mmap将Binder驱动使用的一块内存映射到用户进程中，
* 通过Binder驱动向SM发送请求
* 获得请求的结果

当然，客户端只需要打开一次Binder驱动，执行一次mmap，所有Binder线程共享。

比如使用ActivityManager获取一些信息，会通过ServiceManager.getService(Context.ACTIVITY_SERVICE)获取AMS的Binder句柄。下面来看获取Binder服务端句柄的具体流程：

## 获取ServiceManager的代理
/frameworks/base/core/java/android/os/ServiceManager.java
```
public static IBinder getService(String name) {
    try {
        IBinder service = sCache.get(name);//先查询缓存
        if (service != null) {
            return service;
        } else {
            return Binder.allowBlocking(rawGetService(name));
        }
    } catch (RemoteException e) {
        Log.e(TAG, "error in getService", e);
    }
    return null;
}
```
`rawGetService(name)`会调用到`getIServiceManager().getService(name)`
```
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }
    // Find the service manager
    sServiceManager = ServiceManagerNative
            .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
    return sServiceManager;
}
```
`IServiceManager`是从`ServiceManagerNative.asInterface(IBinder obj)`而来，继续追踪：
```
public abstract class ServiceManagerNative extends Binder implements IServiceManager
{
    static public IServiceManager asInterface(IBinder obj)
    {
        if (obj == null) {
            return null;
        }
        IServiceManager in =
            (IServiceManager)obj.queryLocalInterface(descriptor);
        if (in != null) {
            return in;
        }
        return new ServiceManagerProxy(obj);
    }

    ...
}
```
查询本地是否有IServiceManager，没有的话就创建`ServiceManagerProxy`。这里回看到上面说会调用`getIServiceManager().getService(name)`，看到这里我们知道就是调用了`ServiceManagerProxy.getService(name)`
```
class ServiceManagerProxy implements IServiceManager {
    public ServiceManagerProxy(IBinder remote) {
        mRemote = remote;
    }

    public IBinder asBinder() {
        return mRemote;
    }

    public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        IBinder binder = reply.readStrongBinder();
        reply.recycle();
        data.recycle();
        return binder;
    }

    ...
}
```
这里的`mRemote`（IBinder类型）也就是传入的`BinderInternal.getContextObject()`，ServiceManagerProxy就是代理的它。利用IBinder对象执行命令，GET_SERVICE_TRANSACTION表示获取ServiceManager的Binder句柄。`transact`方法内部会使用ProcessState和IPCThreadState来与Binder驱动通信。

## IBinder和BpBinder
前面获取ServiceManager的代理，是通过一个`IBinder`对象的`transact`方法来执行与Binder驱动通信。现在来看IBinder内部如何完成与Binder驱动通信的任务。


IBinder接口的主要方法如下：

/frameworks/base/core/java/android/os/IBinder.java
```
public interface IBinder {
    public @Nullable IInterface queryLocalInterface(String descriptor);
    public boolean transact(int code, Parcel data, Parcel reply, int flags)
        throws RemoteException;
    ...
```

通过`BinderInternal`获取IBinder：
/frameworks/base/core/java/com/android/internal/os/BinderInternal.java
```
public class BinderInternal {
     /**
     * Return the global "context object" of the system.  This is usually
     * an implementation of IServiceManager, which you can use to find
     * other services.
     */
    public static final native IBinder getContextObject();
    ...
```
这是一个native方法，所以还要找到JNI层的BinderInternal：
/frameworks/base/core/jni/android_util_Binder.cpp
```
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```
&emsp;&emsp;可以看到实际是通过`ProcessState`来实现的，把`ProcessState`创建的C++层的IBinder指针转为了Java层的IBinder对象。`IBinder`只是一个接口，在Native层，是`BpBinder`（BpBinder.cpp）实现了它，而在Java层，是Binder.java中的`BinderProxy`。

&emsp;&emsp;在这里，ProcessState::self()->getContextObject(NULL)返回的就是一个`BpBinder`的对象。然后通过`javaObjectForIBinder()`函数被转为Java层的`BinderProxy`。<br/>
&emsp;&emsp;所以在Java层使用`mRemote.transact()`也就是`BinderProxy.transact()`。而`BinderProxy.transact()`实际还是通过JNI调用到转换前的`BpBinder`。JNI函数也在android_util_Binder.cpp中。

最终还是执行`BpBinder.transact`来处理用户的Binder请求：
/frameworks/native/libs/binder/BpBinder.cpp
```
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```
可以看到是调用了`IPCThreadState.transact`，可以发现与Binder驱动通信，还是依赖于`ProcessState`和`IPCThreadState`。

## ProcessState和IPCThreadState
* 一个进程只有一个`ProcessState`实例，只有在`ProcessState`实例创建时才会打开Binder设备和完成内存映射。
* `ProcessState`用于向上层提供跨进程通信服务
* `ProcessState`和`IPCThreadState`分工合作

Android中启动应用进程，是通过socket和Zygote进程通信。Zygote进程由init进程启动，进程启动之后调用`ZygoteInit.main()`方法，其中会调用`runSelectLoop()`方法，`runSelectLoop()`会创建I/O多路复用，监听socket。收到启动进程的socket请求后，会调用ZygoteInit.java的`acceptCommandPeer()`方法创建ZygoteConnection，然后调用ZygoteConnection的`processOneCommand()`方法，其中会先调用`Zygote.java的forkAndSpecialize()`方法fork出新进程，然后对进程做一些初始化操作：调用ZygoteConnection的`handleChildProc()` --> `ZygoteInit.zygoteInit()` --> `ZygoteInit.nativeZygoteInit()` --> `AndroidRuntime.cpp`的子类`AppRuntime`的`onZygoteInit()`方法。
```
virtual void onZygoteInit() {
    sp<ProcessState> proc = ProcessState::self();
    ALOGV("App process: starting thread pool.\n");
    proc->startThreadPool();
}
```
`onZygoteInit`方法会先调用`ProcessState`的`self()`方法，然后调用其`startThreadPool()`方法。

```
// frameworks/native/libs/binder/ProcessState.cpp
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}
```
> 进程启动就会执行`ProcessState.self()`，而后面跨进程时使用也会调用`ProcessState.self()`，`self()`函数内部保证了进程单例。

`self()`方法中构造了ProcessState实例，构造函数如下：
```
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver))  //打开/dev/binder驱动设备
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        /从binder驱动映射一块内存到应用进程用于数据传输,大小为1M-8k
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
            close(mDriverFD);//失败则关闭
            mDriverFD = -1;
            mDriverName.clear();
        }
    }

    LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
}
```
`ProcessState`构造函数中，先调用`ProcessState.open_driver()`--> open系统调用 --> binder.c的binder_open方法打开/dev/binder驱动设备，再利用`mmap()`映射binder内核驱动的地址空间到应用的地址空间中，用于交互操作。

在前面说的通过`ProcessState::self()->getContextObject(NULL)`返回了`BpBinder`对象
```
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(0);//0就是SM
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;//将被返回的IBinder

    AutoMutex _l(mLock);

    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            if (handle == 0) {
                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;//获取SM状态，如果未存活则返回null
            }

            b = BpBinder::create(handle);//这里看到了BpBinder
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }
    return result;
}
```
从上面代码，可以看到`ProcessState`中有一份列表来记录所有与Binder对象有关的信息，数据类型是`handle_entry`：
```
struct handle_entry {
    IBinder* binder;//对应BpBinder
    RefBase::weakref_type* refs;
};
```

BpBinder::create会执行到`BpBinder`的构造函数：
```
BpBinder::BpBinder(int32_t handle, int32_t trackedUid)
    : mHandle(handle)//如果是SM，则handle为0
    , mAlive(1)
    , mObitsSent(0)
    , mObituaries(NULL)
    , mTrackedUid(trackedUid)
{
    extendObjectLifetime(OBJECT_LIFETIME_WEAK);//弱引用
    IPCThreadState::self()->incWeakHandle(handle, this);
}
```
BpBinder的构造函数中出现了`IPCThreadState`，它的`self()`函数可以保证一个线程只会构造一次。下面看`IPCThreadState`的构造函数：
```
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()), //ProcessState为进程单例
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256); //mIn是一个Pacel，用于接收从Binder发来的数据
    mOut.setDataCapacity(256); //mOut也是Pacel，用于存储要发给Binder的数据
}
```
在这里回头看一下前面，从`ServiceManager`的`getService`开始：`ServiceManagerProxy.getService` -> `BinderProxy.getService` -> `BpBinder.transact` -> `IPCThreadState.transact`

所以最终执行的是`IPCThreadState.transact()`，下面分段分析

#### IPCThreadState.transact()
```
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
```
对于getService：
* handle为0
* code为GET_SERVICE_TRANSACTION
* data

在前面`ServiceManagerProxy.getService`中对data的操作为：
```
data.writeInterfaceToken(IServiceManager.descriptor);
data.writeString(name);//name为要查询的服务名，如 "window"表示WMS
```
* flags为0


```
    status_t err;
    flags |= TF_ACCEPT_FDS;
    ...
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
```
flags有4种:
* TF_ONE_WAY：表示当前业务异步，不阻塞
* TF_ROOT_OBJECT：包含的内容是根对象
* TF_STATUS_CODE：包含的内容是32位的状态值
* TF_ACCEPT_FDS=0x10：允许回复中包含文件描述符

此时的情况，flags开始为0，到这里就是TF_ACCEPT_FDS。`writeTransactionData`将检查data是否有效，并将data包装后的`binder_transaction_data`类型后写入mOut中。这里只是准备数据，实际要在`talkWithDriver()`中才会实际发送给Binder驱动。


```
    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }//出错处理

    if ((flags & TF_ONE_WAY) == 0) {//不是异步
        ...
        if (reply) {//如果reply对象不为空
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;//空的话就发一个假的reply
            err = waitForResponse(&fakeReply);
        }
        ...
    } else {//异步
        err = waitForResponse(NULL, NULL);
    }

    return err;
}
```
到这里，把要发送给Binder的数据按照Binder协议准备好了，但还有以下主要步骤：
* `BC_TRANSACTION`是 `BINDER_WRITE_READ`的子命令，所以mOut中的数据会被再包装一层描述信息
* 还要实际发送数据
* Binder跨进程通信大都是阻塞型，后续将看到如何实现

现在开始看`waitForResponse`方法做了什么：

#### waitForResponse
```
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {//循环等待结果
        if ((err=talkWithDriver()) < NO_ERROR) break;  //这里的talkWithDriver是关键，它会处理和Binder间的交互命令
        err = mIn.errorCheck();//talkWithDriver执行结束，mIn中则接收到Binder驱动给的数据
        if (err < NO_ERROR) break;//出错退出
        if (mIn.dataAvail() == 0) continue;//mIn中没有数据，则继续循环

        cmd = (uint32_t)mIn.readInt32();
        //下面是从Binder接收到的回复数据的处理
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_ACQUIRE_RESULT:
            {
                ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(NULL,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t), this);
                    }
                } else {
                    freeBuffer(NULL,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t), this);
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}
```
`waitForResponse`完成了前面说的三个步骤。`talkWithDriver()`中会包装数据并发给Binder驱动；`talkWithDriver()`函数返回也就是收到了Binder驱动的回复，这意味着Binder Server已经执行了相关请求，所以是否阻塞也是在`talkWithDriver()`中处理。

`talkWithDriver()`执行完后，收到`BR_TRANSACTION_COMPLETE`命令，这里会switch-break，然后执行下一次循环，也就会再次调用`talkWithDriver()`。而这次的bwr.write_size为0，所以在binder_ioctl中，不会执行`binder_thread_write`，而直接执行`binder_thread_read`，其中会进入休眠，等待SM的回复。

在Binder原理2中，知道SM启动后，执行到binder_loop的循环中。

#### talkWithDriver
```
status_t IPCThreadState::talkWithDriver(bool doReceive) //默认参数为true
{
    if (mProcess->mDriverFD <= 0) {//如果Binder驱动还没有打开
        return -EBADF;
    }

    binder_write_read bwr;//读写都用这个数据结果

    // mIn的剩余需要读取的缓存大小是否为0，不为0则需要读
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
```


```   
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();
```
在bwr中填写需要write的内容和大小


```
    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
```
在bwr中填写需要read的内容和大小


```
    ...
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;
```
如果没有需要写也没有需要读的数据就直接返回


```
    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        
#if defined(__ANDROID__)//如果是Android系统
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        ...
    } while (err == -EINTR);//一般只会循环一次
```
这里可以看到与Binder驱动通信的关键：`ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)`

现在可以回到Binder驱动章节，查看ioctl函数的内容。最终Binder驱动添加了t到对方（也就是SM）的处理队列，tcomplete添加到自身的处理队列，然后唤醒Service Manager。然后返回Binder驱动的`binder_thread_write`函数，这里没有什么需要做的，所以再返回到Binder驱动的`binder_thread_write`。
执行完后，返回到这里。

```
    ...
    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        ...
        return NO_ERROR;
    }

    return err;
}
```
执行完`ioctl`后，通过bwr.write_comsumed和bwr.read_comsumed可以知道Binder驱动对请求的BINDER_WRITE_READ命令的处理情况，然后对mOut和mIn做处理：
* bwr.write_consumed > 0 说明Binder驱动消耗了mOut中的数据，所以要把这部分已处理过的数据移除掉。如果消耗的量小于总量，就只删去这部分数据；否则设置总量为0
* bwr.read_consumed > 0 说明Binder驱动回复了数据，写到了mIn.data()指向的地址，则设置dataSize和dataPos为适当值。

```
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```










构造完`ProcessState`后，会调用其`startThreadPool()`方法
```
void ProcessState::startThreadPool()
{
    AutoMutex _l(mLock);  //多线程同步
    if (!mThreadPoolStarted) {  //使用标志变量保证只初始化一次binder线程池
        mThreadPoolStarted = true;
        spawnPooledThread(true);
    }
}
```
```
void ProcessState::spawnPooledThread(bool isMain)
{
    if (mThreadPoolStarted) {
        String8 name = makeBinderThreadName();  //获取Binder线程名
        ALOGV("Spawning new pooled thread, name=%s\n", name.string());
        sp<Thread> t = new PoolThread(isMain);  //isMain为true
        t->run(name.string());
    }
}
```
startThreadPool()中调用`spawnPooledThread(true)`是进程创建时创建binder主线程，后面还会调用`spawnPooledThread(false)`以创建普通binder线程
```
String8 ProcessState::makeBinderThreadName() {
    int32_t s = android_atomic_add(1, &mThreadPoolSeq);
    pid_t pid = getpid();
    String8 name;
    name.appendFormat("Binder:%d_%X", pid, s);
    return name;
}
```
```
//父类Thread位于/system/core/libutils/Threads.cpp
class PoolThread : public Thread
{
public:
    explicit PoolThread(bool isMain)
        : mIsMain(isMain)
    {
    }
    
protected:
    virtual bool threadLoop()
    {
        IPCThreadState::self()->joinThreadPool(mIsMain);
        return false;
    }
    
    const bool mIsMain;
};
```
PoolThread是一个线程，spawnPooledThread中的t->run则会执行它的`threadLoop`方法，返回false就只会执行一次。创建IPCThreadState并执行joinThreadPool方法

```
void IPCThreadState::joinThreadPool(bool isMain)
{
    LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());
    //isMain为true，写入BC_ENTER_LOOPER，会在Binder.c中binder_thread_write方法中用到
    mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);

    status_t result;
    do {
        processPendingDerefs();
        // now get the next command to be processed, waiting if necessary
        result = getAndExecuteCommand();

        if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
            ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
                  mProcess->mDriverFD, result);
            abort();
        }

        // Let this thread exit the thread pool if it is no longer
        // needed and it is not the main process thread.
        if(result == TIMED_OUT && !isMain) {
            break;
        }
    } while (result != -ECONNREFUSED && result != -EBADF);

    LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%d\n",
        (void*)pthread_self(), getpid(), result);

    mOut.writeInt32(BC_EXIT_LOOPER);
    talkWithDriver(false);
}
```
这里线程会死循环，不断调用getAndExecuteCommand()。mIn 用来接收来自Binder设备的数据，mOut用来存储发往Binder设备的数据

```
status_t IPCThreadState::getAndExecuteCommand()
{
    status_t result;
    int32_t cmd;

    result = talkWithDriver();
    if (result >= NO_ERROR) {
        size_t IN = mIn.dataAvail();
        if (IN < sizeof(int32_t)) return result;
        cmd = mIn.readInt32();
        IF_LOG_COMMANDS() {
            alog << "Processing top-level Command: "
                 << getReturnString(cmd) << endl;
        }

        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount++;
        if (mProcess->mExecutingThreadsCount >= mProcess->mMaxThreads &&
                mProcess->mStarvationStartTimeMs == 0) {
            mProcess->mStarvationStartTimeMs = uptimeMillis();
        }
        pthread_mutex_unlock(&mProcess->mThreadCountLock);

        result = executeCommand(cmd);

        pthread_mutex_lock(&mProcess->mThreadCountLock);
        mProcess->mExecutingThreadsCount--;
        if (mProcess->mExecutingThreadsCount < mProcess->mMaxThreads &&
                mProcess->mStarvationStartTimeMs != 0) {
            int64_t starvationTimeMs = uptimeMillis() - mProcess->mStarvationStartTimeMs;
            if (starvationTimeMs > 100) {
                ALOGE("binder thread pool (%zu threads) starved for %" PRId64 " ms",
                      mProcess->mMaxThreads, starvationTimeMs);
            }
            mProcess->mStarvationStartTimeMs = 0;
        }
        pthread_cond_broadcast(&mProcess->mThreadCountDecrement);
        pthread_mutex_unlock(&mProcess->mThreadCountLock);
    }

    return result;
}
```
这里的关键，一是`talkWithDriver()`，二是`executeCommand()`。`talkWithDriver()`方法会读出mIn和mOut，并调用ioctl执行到 binder_ioctl --> binder_ioctl_write_read --> binder_thread_write或binder_thread_read。也就是不断地和Binder通信，并执行命令。

-----------------------------
综上，每个进程启动后，都会调用open_binder打开binder驱动，然后调用mmap映射binder内核空间到应用空间，并循环读写并执行binder相关的命令