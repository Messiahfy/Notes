## yield函数
协程中的大多数函数的作用都好理解，但`yield`函数容易产生疑惑，协程中的`yield`函数在官方注释文档中的说明是，让同一个协程调度器的其他协程先执行。先了解它的源码：
```
public suspend fun yield(): Unit = suspendCoroutineUninterceptedOrReturn sc@ { uCont ->
    val context = uCont.context
    // 先检测协程状态
    context.ensureActive()
    // 调用上下文中的拦截器，调度器继承自CoroutineDispatcher，会返回DispatchedContinuation类型
    val cont = uCont.intercepted() as? DispatchedContinuation<Unit> ?: return@sc Unit
    // 判断调度器是否需要调度，例如Android中的Main会调度，而Main.immediate只要已经在主线程就不需要调度，而Default调度器是肯定返回true
    if (cont.dispatcher.isDispatchNeeded(context)) {
        // 所以这里就是对应Default或者Android中的Main等调度器，调用dispatchYield
        cont.dispatchYield(context, Unit)
    } else {
        // 到这里的话，就是Main.immediate或者Unconfined。
        // YieldContext会在Unconfined内被判断
        val yieldContext = YieldContext()
        // 仍然调用dispatchYield
        cont.dispatchYield(context + yieldContext, Unit)
        // Unconfined的内dispatch（被dispatchYield调用）会设置dispatcherWasUnconfined为true
        if (yieldContext.dispatcherWasUnconfined) {
            // 是Unconfined的情况，判断当前Unconfined对应的eventLoop是否有任务执行，有的话则需要挂起，没有就直接返回
            return@sc if (cont.yieldUndispatched()) COROUTINE_SUSPENDED else Unit
        }
        // Otherwise, it was some other dispatcher that successfully dispatched the coroutine
    }
    COROUTINE_SUSPENDED
}
```

常规情况会调用`dispatchYield`，`dispatchYield`默认就是调用`dispatch`，所以Main或者Main.immediate如果调用了`dispatchYield`，都会使用`handler.post`，这样就将任务放到了最后执行，达到了让其他协程先执行的目的。如果是Default调度器，它继承自ExperimentalCoroutineDispatcher，它的`dispatchYield`如下：
```
override fun dispatchYield(context: CoroutineContext, block: Runnable): Unit =
    try {
        // 这里tailDispatch为true，会把任务放到任务队列的末尾
        coroutineScheduler.dispatch(block, tailDispatch = true)
    } catch (e: RejectedExecutionException) {
        // CoroutineScheduler only rejects execution when it is being closed and this behavior is reserved
        // for testing purposes, so we don't have to worry about cancelling the affected Job here.
        DefaultExecutor.dispatchYield(context, block)
    }
```
`dispatchYield`的作用，通过以上几个例子，可以明白`yield`如何达到让步。

对于`Unconfined`的情况有特殊的判定，`Unconfined`源码如下：
```
internal object Unconfined : CoroutineDispatcher() {
    // isDispatchNeeded总是返回false，所以Unconfined不会调度代码，代码原本在哪个线程，就还是依然在哪个线程执行
    override fun isDispatchNeeded(context: CoroutineContext): Boolean = false

    // 因为isDispatchNeeded返回false，所以常规情况不会执行这个dispatch函数
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        // 仅仅yield函数可以调用这个Unconfined的dispatch，因为yield内部才会设置YieldContext
        val yieldContext = context[YieldContext]
        if (yieldContext != null) {
            // report to "yield" that it is an unconfined dispatcher and don't call "block.run()"
            yieldContext.dispatcherWasUnconfined = true
            return
        }
        // 其他地方使用Unconfined的dispatch都会抛出异常
        throw UnsupportedOperationException("Dispatchers.Unconfined.dispatch function can only be used by the yield function. " +
            "If you wrap Unconfined dispatcher in your code, make sure you properly delegate " +
            "isDispatchNeeded and dispatch calls.")
    }
    
    override fun toString(): String = "Dispatchers.Unconfined"
}
```

`yield`调用了`Unconfined`的`dispatch`，然后判断dispatcherWasUnconfined为true，继续执行`executeUnconfined`：
```
private inline fun DispatchedContinuation<*>.executeUnconfined(
    contState: Any?, mode: Int, doYield: Boolean = false,
    block: () -> Unit
): Boolean {
    assert { mode != MODE_UNINITIALIZED } // invalid execution mode
    // 事件循环，看起来就是用在不需要调度的任务执行的队列，比如Unconfined调度器
    val eventLoop = ThreadLocalEventLoop.eventLoop
    // 如果使用了yield，并且队列为空，因此不需要让步，返回false，yield中就会直接结束，不会挂起
    if (doYield && eventLoop.isUnconfinedQueueEmpty) return false
    return if (eventLoop.isUnconfinedLoopActive) {
        // 如果unconfined loop活跃，unconfined的任务数量大于1，就调度任务避免栈溢出。因为unconfined是直接执行，连续执行次数太多会导致栈过高，所以通过调度来避免。
        _state = contState
        resumeMode = mode
        // 将任务调度到任务队列最后，DispatchedContinuation是一个task，实现的run就是resume，恢复协程的挂起函数
        eventLoop.dispatchUnconfined(this)
        true // 需要挂起
    } else {
        // unconfined loop没有激活，就执行当前block（也就是DispatchedContinuation的run，主要就是resume），和全部unconfinedQueue中的任务，不会挂起
        runUnconfinedEventLoop(eventLoop, block = block)
        false
    }
}
```

判断 unconfined loop 是否活跃，数量大于0x100000000就认为是活跃，因为unconfined的情况，增加count也是按0x100000000增加的
```
public val isUnconfinedLoopActive: Boolean
    get() = useCount >= delta(unconfined = true)

// unconfined的情况，1左移32位，也就是0x100000000
private fun delta(unconfined: Boolean) =
    if (unconfined) (1L shl 32) else 1L

// 增加使用数量
fun incrementUseCount(unconfined: Boolean = false) {
        useCount += delta(unconfined)
        if (!unconfined) shared = true 
    }
```