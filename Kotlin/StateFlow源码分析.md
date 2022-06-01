## StateFlow源码分析
在使用`StateFlow`的时候，一般使用的是`StateFlowImpl`，它继承了几个父类或接口：
* AbstractSharedFlow：提供管理订阅者的方式。和SharedFlow一样继承了它，但使用的slot子类不同
* MutableStateFlow：也是MutableSharedFlow的子类。表示是可变的StateFlow，提供emit方法等
* CancellableFlow：标识flow可以取消
* FusibleFlow：可使用flowOn和buffer改变context或者容量

### 管理订阅者
与SharedFlow不同的是，StateFlow使用的slot子类为`StateFlowSlot`：
```
private class StateFlowSlot : AbstractSharedFlowSlot<StateFlowImpl<*>>() {
    /**
     * 每个Slot可以有如下几种状态:
     *
     * * `null` -- 当前没有使用。可以调用 [allocateLocked] 和新的collector锁定.
     * * `NONE` -- 被一个collector使用，但既没有挂起也没有待处理的值.
     * * `PENDING` -- 待处理新值。
     * * `CancellableContinuationImpl<Unit>` -- 挂起等待新值
     */
    private val _state = atomic<Any?>(null)

    // 新的订阅者申请使用这个SharedFlowSlot，true表示申请成功，false表示失败
    override fun allocateLocked(flow: StateFlowImpl<*>): Boolean {
        // No need for atomic check & update here, since allocated happens under StateFlow lock
        if (_state.value != null) return false // not free
        _state.value = NONE // allocated
        return true
    }

    override fun freeLocked(flow: StateFlowImpl<*>): Array<Continuation<Unit>?> {
        _state.value = null // free now
        return EMPTY_RESUMES // nothing more to do
    }

    @Suppress("UNCHECKED_CAST")
    fun makePending() {
        _state.loop { state ->
            when {
                state == null -> return // this slot is free - skip it
                state === PENDING -> return // already pending, nothing to do
                state === NONE -> { // mark as pending
                    if (_state.compareAndSet(state, PENDING)) return
                }
                else -> { // must be a suspend continuation state
                    // we must still use CAS here since continuation may get cancelled and free the slot at any time
                    if (_state.compareAndSet(state, NONE)) {
                        (state as CancellableContinuationImpl<Unit>).resume(Unit)
                        return
                    }
                }
            }
        }
    }

    fun takePending(): Boolean = _state.getAndSet(NONE)!!.let { state ->
        assert { state !is CancellableContinuationImpl<*> }
        return state === PENDING
    }

    @Suppress("UNCHECKED_CAST")
    suspend fun awaitPending(): Unit = suspendCancellableCoroutine sc@ { cont ->
        assert { _state.value !is CancellableContinuationImpl<*> } // can be NONE or PENDING
        if (_state.compareAndSet(NONE, cont)) return@sc // installed continuation, waiting for pending
        // CAS failed -- the only possible reason is that it is already in pending state now
        assert { _state.value === PENDING }
        cont.resume(Unit)
    }
}
```

### 接收数据
```
override suspend fun collect(collector: FlowCollector<T>) {
    val slot = allocateSlot() // SharedFlow中已分析，每个Collector关联一个slot
    try {
        // flow调用了onSubscription的情况，会使用SubscribedFlowCollector包装原本的collector，在这里回调订阅开始
        if (collector is SubscribedFlowCollector) collector.onSubscription()
        // 获取订阅flow的协程job
        val collectorJob = currentCoroutineContext()[Job]
        var oldState: Any? = null // previously emitted T!! | NULL (null -- nothing emitted yet)
        while (true) {
            // Here the coroutine could have waited for a while to be dispatched,
            // so we use the most recent state here to ensure the best possible conflation of stale values
            val newState = _state.value
            // always check for cancellation
            collectorJob?.ensureActive()
            // Conflate value emissions using equality
            if (oldState == null || oldState != newState) { // 第一次循环将直接emit当前值，再次循环将比较新旧值是否相同
                collector.emit(NULL.unbox(newState))
                oldState = newState
            }
            if (!slot.takePending()) { // 先尝试非挂起方式，判断slot的state是否为PENDING，只有当设置新值后才会为PENDING
                slot.awaitPending() // only suspend for new values when needed
            }
        }
    } finally {
        freeSlot(slot)
    }
}
```

`StateFlowSlot takePending`就是直接判断state是否为PENDING
```
fun takePending(): Boolean = _state.getAndSet(NONE)!!.let { state ->
    assert { state !is CancellableContinuationImpl<*> }
    return state === PENDING
}
```

`StateFlowSlot awaitPending`，如果是NONE状态，则设置为continuation，等待新值；如果是PENDING，则结束挂起，继续上面的循环
```
suspend fun awaitPending(): Unit = suspendCancellableCoroutine sc@ { cont ->
    assert { _state.value !is CancellableContinuationImpl<*> } // can be NONE or PENDING
    if (_state.compareAndSet(NONE, cont)) return@sc // 设置state为continuation
    // CAS failed -- the only possible reason is that it is already in pending state now
    assert { _state.value === PENDING }
    cont.resume(Unit)
}
```

### 发射数据
```
// sequence是全局变量，当数据更新时，sequence的值也会改变。当sequence为奇数时，表示当前数据正在更新，并发更新时可以通过它判断是否正在并发
private var sequence = 0

private fun updateState(expectedState: Any?, newState: Any): Boolean {
    var curSequence = 0
    var curSlots: Array<StateFlowSlot?>? = this.slots // benign race, we will not use it
    synchronized(this) {
        val oldState = _state.value
        if (expectedState != null && oldState != expectedState) return false // CAS support
        if (oldState == newState) return true // Don't do anything if value is not changing, but CAS -> true
        _state.value = newState // 更新值
        curSequence = sequence
        if (curSequence and 1 == 0) { // 偶数表示没有并发
            curSequence++ // 修改sequence为奇数
            sequence = curSequence
        } else {
            // 如果为奇数，说明正在并发修改，修改sequence的值，但不改变奇偶性，仅用于其他并发操作判断当前数据发生了改变
            sequence = curSequence + 2 // change sequence to notify, keep it odd
            return true // 返回true让并发的操作执行到下面即可，因为下面会通知所有Collector，只需要一个线程执行即可
        }
        curSlots = slots // read current reference to collectors under lock
    }
    // 在锁之外通知所有订阅者更新，以避免无约束协程的死锁??
    while (true) {
        // 遍历通知所有订阅者
        curSlots?.forEach {
            it?.makePending()
        }
        // 加锁判断是否在通知的过程中，值又被改变。
        synchronized(this) {
            if (sequence == curSequence) { // 没有改变，则完成
                sequence = curSequence + 1 // 改回偶数
                return true // done, updated
            }
            // 如果通知的过程中，值又被更新了，则再次循环通知
            curSequence = sequence
            curSlots = slots
        }
    }
}
```


awaitPending()等待数据，修改数据后调makePending()通知挂起函数（挂起函数的continuation放在了_state中）
https://tech.bytedance.net/articles/7025875532008931335