## Channel
Channel提供了协程之间的数据通信功能，使用了消费者/生产者模型，类似BlockingQueue，只不过`send`和`receive`都是挂起函数，而不是BlockingQueue那样阻塞线程。
```
// 简单示例
val channel = Channel<Int>()
GlobalScope.launch {
    channel.send(1)
}
GlobalScope.launch {
    channel.receive()
}
```

构造Channel的函数如下：
```
public fun <E> Channel(
    capacity: Int = RENDEZVOUS,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND,
    onUndeliveredElement: ((E) -> Unit)? = null
): Channel<E>
```
capacity是Channel的容量，配置的区别是这样：
* RENDEZVOUS：没有缓存，send后会挂起，直到有receive接收
* CONFLATED：缓存一个元素，有新的元素，就替换掉旧的（相当于使用了BufferOverflow.DROP_LATEST）
* UNLIMITED：不限制缓存区大小
* BUFFERED：多个元素大小的缓存区，具体数量取决于环境属性，默认64
* 自定义正整数

onBufferOverflow：当缓存已满时Channel的行为
* SUSPEND：当buffer满了，send会挂起
* DROP_OLDEST：当buffer满了，可以send，但会丢弃buffer中最旧的元素
* DROP_LATEST：当buffer满了，可以send，但会丢弃buffer中最新的元素

onUndeliveredElement就是使用DROP_OLDEST或DROP_LATEST，丢弃元素时的回调

### send和receive
send和receive都是挂起函数，并且支持取消。如果Channel关闭，send和receive会抛出异常。

> Channel缓冲区分为链表与数组两种实现。因为不同协程中执行send和receive，所以会用到锁。

> 常规方式创建的Channel既是生产者也是消费者。CoroutineScope.produce和CoroutineScope.actor创建的Channel分别对应生产者和消费者。

> Channel是热流，会直接开始发送值，无论是否有消费者。可能会浪费资源，在想订阅后才执行的场景，可以使用Flow

## Flow
挂起函数可以异步的返回单个值，而Flow可以异步多次返回值。Flow和RxJava的设计相似，Flow在冷流，它在调用collect后才会执行，多次collect调用会多次执行，并且是独立的行为，互不干扰。
```
val flow = flow {
    emit(0)
    emit(1)
}
GlobalScope.launch {
    flow.collect {
        println(it)
    }
}
```
这是一个简单的示例，flow函数的lambda是一个挂起函数，所以在里面也可以调用其他挂起函数。Flow的基本使用方式，阅读官方文档即可。

### Flow源码流程
flow函数会创建一个`SafeFlow`：
```
public fun <T> flow(@BuilderInference block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}
```
flow函数的lambda参数将存储在`SafeFlow`中，这里的collectSafely会被`collect`函数间接调用到，所以我们下面看`collect`函数。

从`collect`函数开始：
```
public suspend inline fun <T> Flow<T>.collect(crossinline action: suspend (value: T) -> Unit): Unit =
    collect(object : FlowCollector<T> {
        override suspend fun emit(value: T) = action(value)
    })
```
Flow的扩展函数`collect`可以传入lambda，在lambda内就可以得到数据。扩展函数`collect`会调用Flow的成员函数`collect`，构造一个匿名`FlowCollector`。

接着来看Flow的子类AbstractFlow的成员函数`collect`实现：
```
// AbstractFlow类
public final override suspend fun collect(collector: FlowCollector<T>) {
    val safeCollector = SafeCollector(collector, coroutineContext)
    try {
        collectSafely(safeCollector)
    } finally {
        safeCollector.releaseIntercepted()
    }
}
```
创建了一个`SafeCollector`，构造函数传入了`collect`扩展函数中创建的匿名`FlowCollector`，以及调用collect的协程的上下文。然后调用`collectSafely`，实现就在开始创建的SafeFlow中。

实现内容就是执行flow扩展函数的lambda，也就会执行SafeCollector的`emit`，：
```
override suspend fun emit(value: T) {
    return suspendCoroutineUninterceptedOrReturn sc@{ uCont ->
        try {
            emit(uCont, value) // 继续调用内部的emit函数
        } catch (e: Throwable) {
            // Save the fact that exception from emit (or even check context) has been thrown
            lastEmissionContext = DownstreamExceptionElement(e)
            throw e
        }
    }
}

private fun emit(uCont: Continuation<Unit>, value: T): Any? {
    val currentContext = uCont.context
    currentContext.ensureActive() // 检测当前协程状态
    // This check is triggered once per flow on happy path.
    val previousContext = lastEmissionContext
    // 如果前后上下文发生变化
    if (previousContext !== currentContext) {
        // 检查是否发生异常，以及emit是否在一个协程中调用（ emit不是线程安全的，防止并发）
        checkContext(currentContext, previousContext, value)
    }
    completion = uCont
    // collector为collect扩展函数中创建的匿名FlowCollector
    // 这里的Continuation传的this，也就是SafeCollector
    return emitFun(collector as FlowCollector<Any?>, value, this as Continuation<Unit>) 
}
```
这里的emitFun是强制转换为Function，也就是把挂起函数转换为实际的带有`Continuation`的函数：
```
private val emitFun =
    FlowCollector<Any?>::emit as Function3<FlowCollector<Any?>, Any?, Continuation<Unit>, Any?>
```
目的就是为了指定Continuation为当前的`SafeCollector`(`SafeCollector`继承了`ContinuationImpl`)，否则就自动用当前挂起函数所在协程（也就是调用collect的协程）作为Continuation了。并且，emitFun的collector为collect扩展函数中创建的匿名FlowCollector，所以将执行它的`emit`方法，也就是执行`collect`扩展函数的lambda代码块，也就是我们获取结果的代码。到这里就追踪完了一个大致的流程。

再分析一下`emitFun`，因为它转换为普通函数了，并且Continuation为`SafeCollector`而不是本来的协程，那么就需要了解`SafeCollector`如何将结果resume回本来的协程。`SafeCollector`重写了`invokeSuspend`：
```
override fun invokeSuspend(result: Result<Any?>): Any {
    result.onFailure { lastEmissionContext = DownstreamExceptionElement(it) }
    completion?.resumeWith(result as Result<Unit>)
    return COROUTINE_SUSPENDED
}
```

`SafeCollector`构造父类：
```
ContinuationImpl(NoOpContinuation, EmptyCoroutineContext)
```
因为ContinuationImpl应该是由挂起函数作为协程启动的时候，由编译器构造，而这里是自己主动构造，所以需要自行指定Continuation，而这里使用了NoOpContinuation，也就是个空操作的Continuation。

FlowCollector的挂起函数emit会执行`collect`扩展函数的lambda代码块（也是一个挂起函数），它们执行完后会resume回Continuation，也就是`SafeCollector`，从BaseContinuationImpl的resumeWith看，`SafeCollector`重写的`invokeSuspend`返回COROUTINE_SUSPENDED，因为默认resume的continuation是NoOpContinuation，所以重写就是为了自己来resume回原本的协程。

**那么为什么`SafeCollector`要自己来resume回原本的协程，我认为是为了在每次发射流程，有异常的情况下，插入自定义的异常提示。**

> collect触发emit，collect的lambda又被emit间接引发调用，所以collect和emit挂起函数都是在collect所在协程执行。但可以通过flowOn来修改emit的协程上下文。

### SharedFlow和StateFlow
`SharedFlow`和`StateFlow`都是热流，并且可以有多个订阅者，也就是广播。先看看`SharedFlow`的简单使用示例：
```
val sharedFlow = MutableSharedFlow<Int>()

GlobalScope.launch {
    delay(1000)
    _sharedFlow.emit(0)
    delay(1000)
    _sharedFlow.emit(1)
    delay(1000)
    _sharedFlow.emit(2
}

GlobalScope.launch {
    sharedFlow.collect {
        Log.e("hhhhh", "collect: $it")
    }
}
```
构造一个MutableSharedFlow，然后在协程中emit和collect数据，由于collect会一直挂起接收数据直到取消，所以一般在不同的协程中执行。热流和RxJava的Subject一样，可以在构造flow之外继续发射数据，而冷流只能在构造的时候确定发射的数据。

MutableSharedFlow的构造方式如下：
```
public fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
): MutableSharedFlow<T>
```
* replay：控制订阅时可以收到几个之前的数据，比如配置为0，就相当于RxJava的PublishSubject，只接收订阅后的数据
* extraBufferCapacity：replay之外的缓存容量，和replay之和就是总缓存容量
* onBufferOverflow：发射的数据超过缓存容量时的策略，可以选择挂起、丢弃最新的、丢弃最旧的


StateFlow是SharedFlow的子类，使用方式类似，但仅包含一个值，表示当前的状态。在StateFlow中，通过Any#equals方法来判断前后两个数据是否相等，相等则会忽略。（所以StateFlow相当于是SharedFlow的一种特例，replay为1，extraBufferCapacity为0，并使用distinctUntilChanged()）

在对应UI状态时，适合使用StateFlow。而一次性的事件，例如显示Toast，则适合使用SharedFlow（replay为0）。

StateFlow和SharedFlow的订阅者都可以被取消，方式就是取消当前协程。因为emit中会调用协程上下文的`ensureActive()`函数，如果取消就抛出CancellationException，另外，在flow构造时和collect时的代码中自己的挂起函数支持取消也是会抛出CancellationException。

**使用的建议**
执行collect建议在使用`repeatOnLifecycle`，例如在Activity中：
```
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        flow.collect {
            //...更新UI
        }
    }
}
```
只要当前状态小于STARTED，就会取消collect，在大于等于STARTED，继续collect。

类似的函数有launchWhenStarted，但在UI中使用Flow时**不建议使用**。因为launchWhenStarted只要有一次大于等于Started后，就会执行到collect，并且变为小于Started后也不会取消，等到DESTROYED才会取消，所以在UI不可见但是未销毁时，更新UI浪费资源。例如launchWhenStarted传入的lambda中有多个挂起函数，执行第一个挂起函数的过程中，状态变为小于Started，那么第一个挂起函数resume后，launchWhenStarted内部不会马上执行后续代码，而是放到队列中，等到状态再次变为Started后继续执行。而flow的collect这种会一直执行的情况并不适合。

---------------------
创建SharedFlow和StateFlow，除了使用MutableSharedFlow和MutableStateFlow，还可以使用`shareIn`和`stateIn`这两个Flow的扩展函数，它们可以把冷流转化为热流，但因为返回的是SharedFlow和StateFlow，所以仍然不能在构造之外再次emit新数据，而是为了可以作为广播，目的是可以节省资源，不用多次执行但可以多个订阅者。

`shareIn`的参数含义：
```
public fun <T> Flow<T>.shareIn(
    scope: CoroutineScope,
    started: SharingStarted,
    replay: Int = 0
): SharedFlow<T>
```
* scope：执行flow代码的协程作用域
* started：SharedFlow的启动和终止模式。Eagerly立即执行并且不会自动终止；Lazily当第一次被订阅时开始执行并且不会自动终止；WhileSubscribed则是第一次被订阅时执行，WhileSubscribed中的stopTimeoutMillis参数配置自动终止时间，默认为0表示没有订阅者就立即终止，replayExpirationMillis参数表示终止后多久才清空replay cache。例如stopTimeoutMillis可以在Activity中使用时配置为5秒（大概选一个值就行），可以避免Activity配置变更的短暂时间重复执行Flow。
* replay：略

`stateIn`的参数含义：
```
public fun <T> Flow<T>.stateIn(
    scope: CoroutineScope,
    started: SharingStarted,
    initialValue: T
): StateFlow<T>
```
* scope：执行flow代码的协程作用域
* started：

使用`shareIn`和`stateIn`的时候，建议赋值给属性，而不是在函数中返回，因为在函数中返回可能会被调用多次，创建多个Flow，并没有达到节省资源的目的。

### flowOn的原理
```
public fun <T> Flow<T>.flowOn(context: CoroutineContext): Flow<T> {
    checkFlowContext(context)
    return when {
        context == EmptyCoroutineContext -> this
        this is FusibleFlow -> fuse(context = context) //对应连续调用flowOn的情况
        else -> ChannelFlowOperatorImpl(this, context = context)
    }
}
```
以常见情况，使用一次flowOn并且切换线程，所以看第三种情况ChannelFlowOperatorImpl。调用它的collect：
```
//调用ChannelFlow的collect
override suspend fun collect(collector: FlowCollector<T>): Unit =
    coroutineScope {
        collector.emitAll(produceImpl(this))
    }
```
emitAll会调用emitAllImpl：
```
private suspend fun <T> FlowCollector<T>.emitAllImpl(channel: ReceiveChannel<T>, consume: Boolean) {
    //...
    try {
        while (true) {
            val result = run { channel.receiveCatching() }
            if (result.isClosed) {
                result.exceptionOrNull()?.let { throw it }
                break // returns normally when result.closeCause == null
            }
            emit(result.getOrThrow())
        }
    } ...
}
```
死循环从channel中接收数据，可以猜测是原本的flow数据会发射到这个channel中，然后使用自行emit发射。那么就需要找到原本的数据怎么发射到了这个channel中：
```
public open fun produceImpl(scope: CoroutineScope): ReceiveChannel<T> =
    scope.produce(context, produceCapacity, onBufferOverflow, start = CoroutineStart.ATOMIC, block = collectToFun)
```
这个produce在Channel中已经介绍过，就是返回一个ReceiveChannel，会执行block：
```
internal val collectToFun: suspend (ProducerScope<T>) -> Unit
    get() = { collectTo(it) }

protected override suspend fun collectTo(scope: ProducerScope<T>) =
    flowCollect(SendingCollector(scope))
```
block中就是执行collectTo函数，SendingCollector如下：
```
public class SendingCollector<T>(
    private val channel: SendChannel<T>
) : FlowCollector<T> {
    override suspend fun emit(value: T): Unit = channel.send(value)
}
```
这个SendingCollector会把数据发送到channel中，flowCollect方法如下：
```
override suspend fun flowCollect(collector: FlowCollector<T>) =
    flow.collect(collector)
```
这里就会调用原本的flow，接收它的数据并发送到channel中，而从collectToFun到flowCollect都是执行在produce使用的协程上下文，也就是flowOn传入的协程上下文。

## 11. Select表达式
还处于实验性状态，不做分析。

## 参考
[协程 Flow 最佳实践 | 基于 Android 开发者峰会应用](https://juejin.cn/post/6844904153181847566)
[从 LiveData 迁移到 Kotlin 数据流](https://juejin.cn/post/6979008878029570055)
[Flow 操作符 shareIn 和 stateIn 使用须知](https://juejin.cn/post/6998066384290709518)
[flowOn()如何做到切换协程 ](https://juejin.cn/post/6914802148614242312#heading-9)