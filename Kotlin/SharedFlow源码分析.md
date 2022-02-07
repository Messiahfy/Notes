## SharedFlow源码分析
在使用`SharedFlow`的时候，一般使用的是`SharedFlowImpl`，它继承了几个父类或接口：
* AbstractSharedFlow：提供管理订阅者的方式，StateFlow也继承了它
* MutableSharedFlow：表示是可变的SharedFlow，提供emit方法等
* CancellableFlow：标识flow可以取消
* FusibleFlow：可使用flowOn和buffer改变context或者容量

### 管理订阅者
父类`AbstractSharedFlow`中是如何管理订阅者的？在`AbstractSharedFlow`中存在一个`slots`数组，`slots`数组中存放`AbstractSharedFlowSlot`类型的数据，`AbstractSharedFlowSlot`用于关联一个订阅者。

在`SharedFlowImpl`中使用的是`AbstractSharedFlowSlot`的子类`SharedFlowSlot`：
```
private class SharedFlowSlot : AbstractSharedFlowSlot<SharedFlowImpl<*>>() {
    // 将被处理的数据的索引，-1表示当前空闲，没有和任何订阅者关联，所以可以被新的订阅者关联
    @JvmField
    var index = -1L // current "to-be-emitted" index, -1 means the slot is free now

    // 保存等待新数据发射的continuation，比如订阅的时候还没有数据，就要挂起等待（collect内会死循环获取数据，所以这一次的挂起结束后还会再次挂起获取数据），有了数据就通知这个continuation
    @JvmField
    var cont: Continuation<Unit>? = null // collector waiting for new value

    // 新的订阅者申请使用这个SharedFlowSlot，true表示申请成功，false表示失败
    override fun allocateLocked(flow: SharedFlowImpl<*>): Boolean {
        if (index >= 0) return false // not free
        index = flow.updateNewCollectorIndexLocked() // 更新SharedFlowImpl的minCollectorIndex，会让slot的index为replayIndex
        return true
    }

    // 订阅者释放关联的SharedFlowSlot
    override fun freeLocked(flow: SharedFlowImpl<*>): Array<Continuation<Unit>?> {
        assert { index >= 0 }
        val oldIndex = index
        index = -1L
        cont = null // cleanup continuation reference
        return flow.updateCollectorIndexLocked(oldIndex) // 更新SharedFlowImpl中的index，具体操作分析在后面了解
    }
}
```

简单看完`SharedFlowSlot`的代码，
```
internal abstract class AbstractSharedFlow<S : AbstractSharedFlowSlot<*>> : SynchronizedObject() {
    // 存放Slot的数组，其中的Slot不一定都有关联订阅者，比如已经被释放关联的情况
    protected var slots: Array<S?>? = null // allocated when needed
        private set
    // 订阅者的数量
    protected var nCollectors = 0 // number of allocated (!free) slots
        private set
    // 下一个空闲的slot的索引的预判
    private var nextIndex = 0 // oracle for the next free slot index

    // 订阅者数量对应的flow，可以对外暴露订阅者数量变化的监听
    private var _subscriptionCount: MutableStateFlow<Int>? = null // init on first need

    val subscriptionCount: StateFlow<Int>
        get() = synchronized(this) {
            // allocate under lock in sync with nCollectors variable
            _subscriptionCount ?: MutableStateFlow(nCollectors).also {
                _subscriptionCount = it
            }
        }

    // 创建一个Slot，由子类实现
    protected abstract fun createSlot(): S

    // 创建一个Slot的数组，由子类实现
    protected abstract fun createSlotArray(size: Int): Array<S?>

    // 订阅者申请关联slot
    protected fun allocateSlot(): S {
        // Actually create slot under lock
        var subscriptionCount: MutableStateFlow<Int>? = null
        val slot = synchronized(this) {
            // 先根据容量确定slots数组的初始化和扩容
            val slots = when (val curSlots = slots) {
                // 如果slots数组为null，则创建并赋值给slots
                null -> createSlotArray(2).also { slots = it }
                // 如果当前订阅者数量已达到slots数组上限，则扩容数组为双倍大小
                else -> if (nCollectors >= curSlots.size) {
                    curSlots.copyOf(2 * curSlots.size).also { slots = it }
                } else {
                    // 如果slots数组容量足够，则不做操作
                    curSlots
                }
            }
            var index = nextIndex
            var slot: S
            // 循环遍历
            while (true) {
                // 如果slots数组中预判的空闲位置的slot为null，则创建新的slot
                slot = slots[index] ?: createSlot().also { slots[index] = it }
                index++
                if (index >= slots.size) index = 0
                // 如果这个slot可以被订阅者关联，则完成操作
                if ((slot as AbstractSharedFlowSlot<Any>).allocateLocked(this)) break // break when found and allocated free slot
            }
            // 更新预判下个空闲slot的位置
            nextIndex = index
            // 订阅者数量增大
            nCollectors++
            subscriptionCount = _subscriptionCount // retrieve under lock if initialized
            slot
        }
        // increments subscription count
        subscriptionCount?.increment(1) // 订阅者数量的flow的值增大
        return slot
    }

    // 订阅者申请释放slot（一般在协程取消或其他异常时才会释放）
    protected fun freeSlot(slot: S) {
        // Release slot under lock
        var subscriptionCount: MutableStateFlow<Int>? = null
        val resumes = synchronized(this) {
            // 订阅者数量减小
            nCollectors--
            subscriptionCount = _subscriptionCount // retrieve under lock if initialized
            // Reset next index oracle if we have no more active collectors for more predictable behavior next time
            if (nCollectors == 0) nextIndex = 0
            // 调用slot的释放
            (slot as AbstractSharedFlowSlot<Any>).freeLocked(this)
        }
        /*
           Resume suspended coroutines.
           This can happens when the subscriber that was freed was a slow one and was holding up buffer.
           When this subscriber was freed, previously queued emitted can now wake up and are resumed here.
        */
        // 这段注释的场景，应该是当前处理数据比较慢的订阅者，结束处理后，其他调用emit但挂起等待的地方可以继续执行
        for (cont in resumes) cont?.resume(Unit)
        // 减少订阅者的数量
        subscriptionCount?.increment(-1)
    }

    // 提供对slot数组的遍历
    protected inline fun forEachSlotLocked(block: (S) -> Unit) {
        if (nCollectors == 0) return
        slots?.forEach { slot ->
            if (slot != null) block(slot)
        }
    }
}
```

### 数据缓存管理
SharedFlowImpl中有一个缓存数组，用于保存emit方法发射数据，数组分成了buffered values和queued emitters两部分（对照源码的注释）
```
/*
    缓存的逻辑结构

              buffered values
         /-----------------------\
                      replayCache      queued emitters
                      /----------\/----------------------\
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
     |   | 1 | 2 | 3 | 4 | 5 | 6 | E | E | E | E | E | E |   |   |   |
     +---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+---+
           ^           ^           ^                      ^
           |           |           |                      |
          head         |      head + bufferSize     head + totalSize
           |           |           |
 index of the slowest  |    index of the fastest
  possible collector   |     possible collector
           |           |
           |     replayIndex == new collector's index
           \---------------------- /
      range of possible minCollectorIndex

      head == minOf(minCollectorIndex, replayIndex) // by definition
      totalSize == bufferSize + queueSize // by definition

   INVARIANTS:
      minCollectorIndex = activeSlots.minOf { it.index } ?: (head + bufferSize)
      replayIndex <= head + bufferSize
 */
```
* buffered values：表示当前缓存数据的大小，最大为replayCache和extraBuffer之和
* queued emitters：当调用emit方法发射数据时，如果缓存数组的buffered values还没有达到容量上限，则发射的数据将保存到缓存中，emit方法会立即返回。如果buffered values已达到容量上限，则调用emit方法会挂起，并且它的continuation和实际数据会被封装成一个Emitter类型的对象，保存到queued emitters部分中。当buffered values中数据被所有的订阅者处理后，才会开始处理queued emitters中Emitter，将其中保存的数据存放到位置n，同时恢复其中保存的emit方法所在续体的执行。之后，位置n将作为buffered values的一部分。？？？？

最慢的订阅者，也就是下游处理最慢的，下游每次处理完后才会再获取下一个数据，还没有获取的时候，数据就还处于缓存中。处理最快的订阅者，也就是把queued emitters的第一个Emitter的数据取出处理并恢复emit方法。

对应源码中的注释来看
```
// 缓存数组，保存了缓存的数据以及挂起情况对应的Emitter数据
private var buffer: Array<Any?>? = null
// replayCache部分的第一个位置，新的订阅者会从这里开始处理数据
private var replayIndex = 0L
// 处理数据最慢的订阅者对应的位置最小的索引
// 如果没有订阅者或者没有额外缓存，则minCollectorIndex的值等于replayIndex
private var minCollectorIndex = 0L
// 缓存数组中buffered values部分的实际数量，和bufferCapacity不同的是，bufferCapacity是最大容量
private var bufferSize = 0
// 缓存数组中queued emitters部分的数量
private var queueSize = 0

// 当前缓存数组的起始位置
private val head: Long get() = minOf(minCollectorIndex, replayIndex)
// 当前缓存数组中replayCache部分的数量
private val replaySize: Int get() = (head + bufferSize - replayIndex).toInt()
// 当前缓存数组中总共缓存数据的数量，包括普通缓存和用于超过缓存容量而挂起的queued emitters
private val totalSize: Int get() = bufferSize + queueSize
// 当前缓存数组中buffered values的末尾位置的后一位
private val bufferEndIndex: Long get() = head + bufferSize
// 当前数组中queued emitters的末尾位置的后一位
private val queueEndIndex: Long get() = head + bufferSize + queueSize
```
buffer数组扩容后，即使只会数据量变小了，它的容量也不会缩小，只是各个index的数据可能为null。buffer的大小取决于缓存数据最多的瞬间申请扩容的大小；head和其他index以及slot的index在使用过程中会不断增大，但因为会和buffer的size - 1 做与运算，所以即使index一直增大，超过buffer的大小，也会在0到size - 1循环操作，猜测这里的意图是为了避免频繁移动数据的位置。

### 接收数据
```
override suspend fun collect(collector: FlowCollector<T>) {
    val slot = allocateSlot() // 前文的AbstractSharedFlow中已介绍
    try {
        // flow调用了onSubscription的情况，会使用SubscribedFlowCollector包装原本的collector，在这里回调订阅开始
        if (collector is SubscribedFlowCollector) collector.onSubscription()
        // 获取订阅flow的协程的Job
        val collectorJob = currentCoroutineContext()[Job]
        while (true) {
            var newValue: Any?
            while (true) {
                // 尝试用非挂起的方式获取数据
                newValue = tryTakeValue(slot) // attempt no-suspend fast path first
                // 如果获取到了，则跳出内部循环
                if (newValue !== NO_VALUE) break
                // 如果非挂起没有获取到数据，则挂起等待数据，等到有数据后会再循环尝试非挂起方式获取
                awaitValue(slot) // await signal that the new value is available
            }
            // 获取到数据，确定协程状态
            collectorJob?.ensureActive()
            // 发射数据给订阅者，然后再进入循环
            collector.emit(newValue as T)
        }
    } finally {
        // 发生取消、异常，结束订阅，释放slot
        freeSlot(slot)
    }
}
```

分析一下`tryTakeValue`函数中非挂起获取数据的流程：
```
private fun tryTakeValue(slot: SharedFlowSlot): Any? {
    var resumes: Array<Continuation<Unit>?> = EMPTY_RESUMES
    val value = synchronized(this) {
        // 如果slot的index在缓存数组中有数据，会返回正常的大于等于0的index
        val index = tryPeekLocked(slot)
        if (index < 0) {
            NO_VALUE
        } else {
            // 有数据的情况：
            val oldIndex = slot.index
            // 从缓存中获取数据
            val newValue = getPeekedValueLockedAt(index)
            // 得到数据后，slot的index指向下一个位置
            slot.index = index + 1 // points to the next index after peeked one
            // 更新缓存数组中的位置信息，并得到Continuation数组
            resumes = updateCollectorIndexLocked(oldIndex)
            newValue
        }
    }
    // 回调Continuation数组
    for (resume in resumes) resume?.resume(Unit)
    return value
}
```

`tryPeekLocked`函数内部的判定方式：
```
// returns -1 if cannot peek value without suspension
private fun tryPeekLocked(slot: SharedFlowSlot): Long {
    // return buffered value if possible
    val index = slot.index
    // 数据的位置处于非挂起的缓存范围内，也就是buffered values，直接返回
    if (index < bufferEndIndex) return index
    // 到这里，是准备获取queued emitters的数据，但因为bufferCapacity大于0，也就是buffered values存在的话，最多只能处理到buffered values的最后一个数据，等待其他订阅者处理数据后更新bufferEndIndex，
    // 使queued emitters中的数据变为buffered values的数据，才可以继续处理
    if (bufferCapacity > 0) return -1L // if there's a buffer, never try to rendezvous with emitters
    // 到这里，是准备获取queued emitters的数据，并且buffered values部分大小为0，只存在queued emitters，这时只能处理第一个数据，要处理下一个就需要挂起
    if (index > head) return -1L // ... but only with the first emitter (never look forward)
    if (queueSize == 0) return -1L // 没有可以处理的 nothing there to rendezvous with
    // 处理queued emitters的第一个数据
    return index // rendezvous with the first emitter
}
```
总结就是，如果有buffered values，就只能处理buffered values中的数据，这种情况，快的订阅者最多可以处理完buffered values的数据，但queued emitters的数据必须挂起等待其他慢的订阅者处理了数据更新bufferEndIndex后，queued emitters范围的数据变为buffered values范围的数据后才可以被处理。没有buffered values的情况，只能处理queued emitters的第一个数据，也就是说各个订阅者会一起处理完一个数据后再一起处理一下个，如果一个订阅者处理较快，也只能挂起等待其他订阅者处理完这个数据后才能接收下一个数据。

得到 >= 0 的index后，从buffer数组中获取该数据，普通数据就直接返回，如果是包装的Emitter，就拿出其中的数据：
```
private fun getPeekedValueLockedAt(index: Long): Any? =
    when (val item = buffer!!.getBufferAt(index)) {
        is Emitter -> item.value
        else -> item
    }


// 这里做了与运算，所以无论index多大，最终都是在数组的size范围内。比如size为8，那么0到7的数和 size - 1 也就是0b0111做与运算，还是本来的数，从8开始与运算结果又会从0开始
private fun Array<Any?>.getBufferAt(index: Long) = get(index.toInt() and (size - 1))
private fun Array<Any?>.setBufferAt(index: Long, item: Any?) = set(index.toInt() and (size - 1), item)
```

获取到数据的情况，需要更新响应的各个index，并返回Continuation的数组用于恢复各个挂起函数的执行：
```
internal fun updateCollectorIndexLocked(oldIndex: Long): Array<Continuation<Unit>?> {
    assert { oldIndex >= minCollectorIndex }
    // 不是最慢的订阅者处理数据，则不需要关心，只需要最慢的那个订阅者处理了数据后执行到这里去更新即可
    if (oldIndex > minCollectorIndex) return EMPTY_RESUMES // nothing changes, it was not min
    // 准备更新最慢的订阅者处理的数据的位置
    val head = head
    var newMinCollectorIndex = head + bufferSize // 先赋值为queued emitters部分的第一个位置
    // 如果只有queued emitters部分，也就是普通的缓存容量为0，那么 newMinCollectorIndex 指向下一个
    if (bufferCapacity == 0 && queueSize > 0) newMinCollectorIndex++
    // newMinCollectorIndex更新为所有关联了订阅者的slot中index最小的那个，主要针对有buffered values的情况
    forEachSlotLocked { slot ->
        @Suppress("ConvertTwoComparisonsToRangeCheck") // Bug in JS backend
        if (slot.index >= 0 && slot.index < newMinCollectorIndex) newMinCollectorIndex = slot.index
    }
    assert { newMinCollectorIndex >= minCollectorIndex } // can only grow
    if (newMinCollectorIndex <= minCollectorIndex) return EMPTY_RESUMES // nothing changes

    var newBufferEndIndex = bufferEndIndex // var to grow when waiters are resumed
    val maxResumeCount = if (nCollectors > 0) {
        // bufferCapacity - newBufferSize0 也就是 bufferCapacity + newMinCollectorIndex - bufferEndIndex，也就是bufferCapacity还剩余的空间
        // 这里的最小值，也就是说queued emitters的数量小于非挂起的剩余容量，就会恢复全部emit挂起函数，反之，只恢复非挂起剩余空间大小内的。
        // 实质上queued emitters中的这部分数据被转化为buffered values内的数据了（在后面几行代码中会更新index），所以需要恢复
        minOf(queueSize, bufferCapacity - newBufferSize0)
    } else {
        // 没有订阅者的情况，因为没有订阅者就直接抛弃数据，那么必须恢复所有挂起的emit函数，所以maxResumeCount为queued emitters的数据数量
        queueSize // that's how many waiting emitters we have (at most)
    }
    var resumes: Array<Continuation<Unit>?> = EMPTY_RESUMES
    val newQueueEndIndex = newBufferEndIndex + queueSize
    if (maxResumeCount > 0) { // collect emitters to resume if we have them
        resumes = arrayOfNulls(maxResumeCount)
        var resumeCount = 0
        val buffer = buffer!!
        for (curEmitterIndex in newBufferEndIndex until newQueueEndIndex) {
            val emitter = buffer.getBufferAt(curEmitterIndex)
            if (emitter !== NO_VALUE) {
                emitter as Emitter // must have Emitter class
                // 存储需要恢复的emitter的continuation到resumes数组中
                resumes[resumeCount++] = emitter.cont
                buffer.setBufferAt(curEmitterIndex, NO_VALUE) // make as canceled if we moved ahead
                // 拆箱数据
                buffer.setBufferAt(newBufferEndIndex, emitter.value)
                newBufferEndIndex++
                if (resumeCount >= maxResumeCount) break // enough resumed, done
            }
        }
    }
    // 得到新的buffered values部分的数量
    val newBufferSize1 = (newBufferEndIndex - head).toInt()
    // 如果没有订阅者，需要恢复所有emit，所以直接让 newMinCollectorIndex 更新为buffered values部分的末尾即可
    if (nCollectors == 0) newMinCollectorIndex = newBufferEndIndex
    // 计算replayIndex的新位置。replay为设定的replay最大容量，取它和buffered values实际数量的较小值，newBufferEndIndex减去它只要比旧的replayIndex更靠后，就需要更新replayIndex
    var newReplayIndex = maxOf(replayIndex, newBufferEndIndex - minOf(replay, newBufferSize1))
    // buffered values不存在的情况，排查buffer中的NO_VALUE数据，调整newBufferEndIndex和newReplayIndex
    if (bufferCapacity == 0 && newReplayIndex < newQueueEndIndex && buffer!!.getBufferAt(newReplayIndex) == NO_VALUE) {
        newBufferEndIndex++
        newReplayIndex++
    }
    // 更新各个index
    updateBufferLocked(newReplayIndex, newMinCollectorIndex, newBufferEndIndex, newQueueEndIndex)
    // just in case we've moved all buffered emitters and have NO_VALUE's at the tail now
    cleanupTailLocked()
    // 现在resumes数组存储的是可以处理的emitter对应的continuation，findSlotsToResumeLocked中会把可以处理的slot的continuation也放进去
    if (resumes.isNotEmpty()) resumes = findSlotsToResumeLocked(resumes)
    return resumes // tryTakeValue中就会恢复这里面所有的continuation对应的挂起函数
}
```

这个函数，内部会把可以处理（条件可见前文的`tryPeekLocked`函数的分析）的slots的continuation一起放到这个resumesIn数组中，然后返回，slot的continuation用于恢复collect内部的awaitValue。`updateCollectorIndexLocked`中调用这个函数，resumesIn包含了参数是所有转化到buffered values的emitter里面的continuation的数组，这样的情况，返回就是emitters和slots的所有continuation数组，emitters的continuation用于恢复挂起的emit
```
private fun findSlotsToResumeLocked(resumesIn: Array<Continuation<Unit>?>): Array<Continuation<Unit>?> {
    var resumes: Array<Continuation<Unit>?> = resumesIn
    var resumeCount = resumesIn.size
    forEachSlotLocked loop@{ slot ->
        val cont = slot.cont ?: return@loop // only waiting slots
        if (tryPeekLocked(slot) < 0) return@loop // only slots that can peek a value
        if (resumeCount >= resumes.size) resumes = resumes.copyOf(maxOf(2, 2 * resumes.size))
        resumes[resumeCount++] = cont
        slot.cont = null // not waiting anymore
    }
    return resumes
}
```

不能以非挂起的形式获取数据的话，就挂起等待通知：
```
private suspend fun awaitValue(slot: SharedFlowSlot): Unit = suspendCancellableCoroutine { cont ->
    synchronized(this) lock@{
        // 再次尝试
        val index = tryPeekLocked(slot) // recheck under this lock
        // 还是小于0，就把continuation存到slot中
        if (index < 0) {
            slot.cont = cont // Ok -- suspending
        } else {
            // 如果拿到了数据，则非挂起完成
            cont.resume(Unit) // has value, no need to suspend
            return@lock
        }
        // 把continuation存到slot中
        slot.cont = cont // suspend, waiting
    }
}
```
这里把continuation放到了slot中，`findSlotsToResumeLocked`函数中会拿到这个continuation去调用resume，从而结束`awaitValue`函数的挂起，再次调用`tryTakeValue`尝试以非挂起的方式获取数据。`findSlotsToResumeLocked`会有多处被调用，主要是发射数据的情况。

### 发射数据
发射数据时，调用的是`emit`函数：
```
override suspend fun emit(value: T) {
    // 先调用tryEmit尝试以非挂起的方式发射数据
    if (tryEmit(value)) return // fast-path
    // 挂起的方式发射数据
    emitSuspend(value)
}
```

非挂起的方式发射数据，调用`tryEmit`：
```
override fun tryEmit(value: T): Boolean {
    var resumes: Array<Continuation<Unit>?> = EMPTY_RESUMES
    // 使用锁，线程安全
    val emitted = synchronized(this) {
        // 尝试发射数据，成功的话会得到slots对应的continuation数组，用于恢复订阅者调用collect中的awaitValue
        if (tryEmitLocked(value)) {
            resumes = findSlotsToResumeLocked(resumes)
            true
        } else {
            false
        }
    }
    // 恢复订阅者的collect中的awaitValue
    for (cont in resumes) cont?.resume(Unit)
    return emitted
}
```

尝试发射数据，调用的是`tryEmitLocked`：
```
private fun tryEmitLocked(value: T): Boolean {
    // 没有订阅者的情况，调用tryEmitNoCollectorsLocked来处理
    if (nCollectors == 0) return tryEmitNoCollectorsLocked(value) // always returns true
    // 有订阅者，并且缓存容量已达上限
    if (bufferSize >= bufferCapacity && minCollectorIndex <= replayIndex) {
        when (onBufferOverflow) {
            // 挂起的情况，返回false
            BufferOverflow.SUSPEND -> return false // will suspend
            // 丢弃最新的数据的情况，返回true
            BufferOverflow.DROP_LATEST -> return true // just drop incoming
            // 丢弃最旧的数据的情况，这里暂不处理
            BufferOverflow.DROP_OLDEST -> {} // force enqueue & drop oldest instead
        }
    }
    // buffered values还可以添加数据，或者已达上限但使用的策略是DROP_OLDEST，此时将数据放到缓存数组的最后
    enqueueLocked(value)
    // 缓存数量加1
    bufferSize++ // value was added to buffer
    // 如果超过容量，就丢弃最旧数据
    if (bufferSize > bufferCapacity) dropOldestLocked()
    // 如果replayCache超出最大容量，就将replayIndex加1，也就是减小了replaySize
    if (replaySize > replay) { // increment replayIndex by one
        updateBufferLocked(replayIndex + 1, minCollectorIndex, bufferEndIndex, queueEndIndex)
    }
    return true
}
```

没有订阅者，将会调用`tryEmitNoCollectorsLocked`：
```
private fun tryEmitNoCollectorsLocked(value: T): Boolean {
    assert { nCollectors == 0 }
    // 不需要replay，则直接丢弃数据，返回true
    if (replay == 0) return true // no need to replay, just forget it now
    // 如果有replay，就把数据放到缓存数组
    enqueueLocked(value) // enqueue to replayCache
    // 已缓存数量加1
    bufferSize++ // value was added to buffer
    // 如果缓存数量超过了replay，就丢弃最旧的
    if (bufferSize > replay) dropOldestLocked()
    // 更新minCollectorIndex
    minCollectorIndex = head + bufferSize // a default value (max allowed)
    return true
}
```

挂起的方式发射数据，调用`emitSuspend`：
```
private suspend fun emitSuspend(value: T) = suspendCancellableCoroutine<Unit> sc@{ cont ->
    var resumes: Array<Continuation<Unit>?> = EMPTY_RESUMES
    // 加锁，线程安全
    val emitter = synchronized(this) lock@{
        // 在此检测是否可以非挂起的方式发射数据
        if (tryEmitLocked(value)) {
            cont.resume(Unit)
            resumes = findSlotsToResumeLocked(resumes)
            return@lock null
        }
        // 到这里就是需要挂起的情况，把数据和continuation包装为一个emitter，并放到缓存数组中
        Emitter(this, head + totalSize, value, cont).also {
            enqueueLocked(it)
            // queued emitters的数量加1
            queueSize++ // added to queue of waiting emitters
            // 如果没有buffered values，就需要获取continuation用于恢复订阅者的awiatValue挂起函数。
            if (bufferCapacity == 0) resumes = findSlotsToResumeLocked(resumes)
        }
    }
    // 注册协程的取消，取消时会调用Emitter的dispose函数，让从缓存数组中清除此Emitter
    emitter?.let { cont.disposeOnCancellation(it) }
    // 恢复订阅者，订阅者继续循环获取数据
    for (r in resumes) r?.resume(Unit)
}
```