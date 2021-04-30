### 进程启动到打开Binder驱动
**以Android 9源码为例：**

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
// frameworks/native/libs/binder/IPCThreadState.cpp

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
        //...

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

```
status_t IPCThreadState::talkWithDriver(bool doReceive) //默认为true
{
    //...

    binder_write_read bwr; // 准备发送给binder驱动的数据格式

    // 是否需要读
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail; // 要写入的数据大小
    bwr.write_buffer = (uintptr_t)mOut.data(); // 要写入的数据

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity(); // 要读取的数据大小
        bwr.read_buffer = (uintptr_t)mIn.data(); // 存储读取的数据
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    //...
    // 不需要读写就直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        //...
#if defined(__ANDROID__)
        // 写入数据到Binder驱动的关键就是这里的ioctl
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        //...
    } while (err == -EINTR);

    //...

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) { //成功写入了数据
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed); // 移除已经被Binder驱动读取的数据
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) { //成功读取到了数据 
            // 修正数据
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        //...
        return NO_ERROR;
    }

    return err;
}
```

-----------------------------
综上，每个进程启动后，都会调用open_binder打开binder驱动，然后调用mmap映射binder内核空间到应用空间，并循环读写并执行binder相关的命令