##  1. 核心概念
RxJava需要先分清楚三个流程，一是链式调用流程，二是订阅流程，三是发布流程。

链式调用：构造出层层包装的Observable，每一步Observable都会包含前一步的Observable。

订阅流程：链式调用最后一步调用subscribe后，从后往前依次调用前一步Observable的subscribeActual方法，通过一层层的subscribeActual就会把一层层包装后的observer传到第一步

发布流程：第一步的subscribeActual，即ObservableCreate的subscribeActual内会开始发布数据，传给Observer，Observer则会层层从前往后传递。

## 2. 核心源码分析
下面是一个RxJava的常见使用流程示例，创建了一个`Observable` ，然后调用`subscribeOn`、`map`、`observeOn`操作符，最终订阅给一个`Observer`。通过这个典型的使用方式，来了解RxJava的相关原理。
```
Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("1");
        emitter.onComplete();
    }
})
        .map(new Function<String, Integer>() {
            @Override
            public Integer apply(String s) throws Exception {
                return Integer.parseInt(s);
            }
        })
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(Disposable d) {

            }
            
            @Override
            public void onNext(Integer integer) {

            }
            
            @Override
            public void onError(Throwable e) {

            }
            
            @Override
            public void onComplete() {

            }
        });
```
### 第一步：create
```
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");//判空
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
`RxJavaPlugins.onAssembly` 是一个用于hook的函数，没有调用 `setOnObservableAssembly` 设置hook的话，就直接返回，这里可以看作直接返回 `ObservableCreate`。

```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
    ...
```
`ObservableCreate` 包装了 `ObservableOnSubscribe`，在订阅流程中，会调到 `subscribeActual` 方法，此时会把 `CreateEmitter`传给 `ObservableOnSubscribe`的 `subscribe` 方法，也就是我们在 `Observable.create`中的匿名内部类 `subscribe` 方法。在匿名 `ObservableOnSubscribe` 内部类的 `subscribe` 方法中我们调用了 `onNext` 等方法，接着就会传给下游的观察者。

### 第二步：map
```
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```
类似地，返回的是 `ObservableMap`，我们写的匿名 `Function` 会传入。
```
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
        source.subscribe(new MapObserver<T, U>(t, function));
    }

    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != NONE) {
                downstream.onNext(null);
                return;
            }

            U v;

            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
            downstream.onNext(v);
        }

        ...
    }
}
```
订阅流程中 `subscribeActual` 就把包装了 `Function` 的 `Observer` 传给第一步的 `Observable`。发布流程中，第一步的 `Observable` 发布数据就会经过这个
 `Observer` ，其中就会在 `onNext` 的时候执行 `Function` 的 `apply`函数，然后再将转换后的结果往后发布。

### 第三步：subscribeOn
```
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
返回一个包含了前一步的 `Observable` 和传入的 `Scheduler` 的 `ObservableSubscribeOn` 对象。
```
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

        observer.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
    ...
```
构造函数也就是在链式调用过程中执行，设置好了前一步的 `Observable` 和传入的 `Scheduler`。然后在订阅流程中，执行 `subscribeActual`，其中会通过 `scheduler.scheduleDirect` 调度到指定线程中执行 `SubscribeTask`。
```
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;
    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }
    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```
可以看到调度线程后执行 `SubscribeTask` 实际就是把继续完成订阅流程，往前一步继续执行订阅。之前的所有订阅流程都会执行在当前 `subscribeOn` 指定的线程中，除非更前面还有调用 `subscribeOn`。注意这里指定的是订阅流程的执行线程，而非发布流程。因为发布流程是由订阅流程执行过来的，如果发布流程未使用 `observeOn` 调度线程，那么发布流程也是执行在订阅流程的线程中。

> 多次调用 `subscribeOn`，`create` 和 `map` 都会执行在最前面的 `subscribeOn` 调度的线程中，即使 `map` 在 `subscribeOn` 之后调用。这并不是说 `subscribeOn` 只有第一次生效。实际上，`subscribeOn` 每次调用都会生效，多次链式调用实质就会导致订阅流程中的代码被层层包裹于不同的线程中。而 `create` 和 `map` 中我们写的代码都是执行在发布流程的，发布流程的代码都是由最内部的订阅流程 `create` 往外层层调用各层的 `Observer`，所以多个 `map` 都会执行在 `create` 所在线程，表现出来就是发布流程只会受到第一个 `subscribeOn` 的影响，而订阅流程会受到每次 `subscribeOn` 影响。

### 第四步：observeOn
```
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, false, bufferSize());
}

public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
返回的是 `ObservableObserveOn` 对象

```
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    final boolean delayError;
    final int bufferSize;
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        super(source);
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        if (scheduler instanceof TrampolineScheduler) {
            source.subscribe(observer);
        } else {
            Scheduler.Worker w = scheduler.createWorker();

            source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
        }
    }
    ...
```
`TrampolineScheduler` 是一个调度在当前线程的调度器，忽略。继续看 `ObserveOnObserver`，因为要支持背压，所以 `ObserveOnObserver` 里面有队列和缓存大小的控制，下面仅列出关键代码。
```
static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {
        ...

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
        }

        @Override
        public void onError(Throwable t) {
            ...
            schedule();
        }

        @Override
        public void onComplete() {
            ...
            schedule();
        }

        void schedule() {
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }

        void drainNormal() {
            int missed = 1;

            final SimpleQueue<T> q = queue;
            final Observer<? super T> a = downstream;

            for (;;) {
                if (checkTerminated(done, q.isEmpty(), a)) {
                    return;
                }

                for (;;) {
                    boolean d = done;
                    T v;

                    try {
                        v = q.poll();
                    } catch (Throwable ex) {
                        Exceptions.throwIfFatal(ex);
                        disposed = true;
                        upstream.dispose();
                        q.clear();
                        a.onError(ex);
                        worker.dispose();
                        return;
                    }
                    boolean empty = v == null;

                    if (checkTerminated(d, empty, a)) {
                        return;
                    }

                    if (empty) {
                        break;
                    }

                    a.onNext(v);
                }

                missed = addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }

        @Override
        public void run() {
            if (outputFused) {
                //少数情况
                drainFused();
            } else {
                drainNormal();
            }
        }

        ...
    }
```
关键是执行 `schedule()`，会调用到指定线程中执行 `run()`，考虑一般情况，执行 `drainNormal()`，其中就会从队列中取出数据发布给下一步的 `Observer`。源线程通过 `onNext` 往队列放数据，而在调度后的线程里面不断从队列里取数据。

> 注意：`observeOn` 控制的是它**之后**的发布流程执行线程，多次调用 `observeOn` 可以改变多次后续步骤中属于**发布流程**的执行所在线程，比如 `map` 等操作符。

### 第五步：subscribe
最后的 `subscribe` 就是订阅流程的起点，从它开始依次调用前一步 `Observable` 的 `subscribeActual` 方法。

### 关于 Disposable
调用 `onSubscribe` 中的 `Disposable` 的 `dispose()` 方法，可以使上游不再调用下游观察者，或者说不再将数据/事件传给下游。

在`ObservableCreate`的`subscribeActual`中会调用`Observer`的`onSubscribe`，然后每个`Observer`在`onSubscribe`也会调用下一个`Observer`的`onSubscribe`，每一层的 `Observer` 都是一个 `Disposable` 对象，经过层层包装，每个`Observer`都会持有前一个`Observer`，调用最后得到的 `Disposable` 的 `dispose()` 方法，就会从后往前调用依次调用每一层 `Observer` 的 `dispose()` 方法，例如 `map` 操作符对应的 `Observer` 仅仅是继续调用前一步的 `dispose()`，`subscribeOn` 和 `observeOn` 除了继续调用前一步的 `dispose()`，还会调整自己的工作状态。

## 3. 常用操作符

### flatmap
注意点：在 `flatMap` 方法参数 `Function` 对象的 `apply` 方法中返回的 `Observable` 末尾加上 `observeOn`来调度线程，行为会比较困惑。

例如：
```
Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("1");
                emitter.onComplete();
            }
        })
                .flatMap(new Function<Integer, ObservableSource<Integer>>() {
                    @Override
                    public ObservableSource<Integer> apply(Integer integer) throws Exception {
                        return Observable.create(new ObservableOnSubscribe<Integer>() {
                            @Override
                            public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
                                emitter.onNext(5);
                                emitter.onComplete();
                            }
                        }).observeOn(Schedulers.io());
                    }
                })....省略
```
`flatMap` 参数内返回的 `Observable` 末尾调用了 `observeOn`，但并不会生效。因为`observeOn`操作符返回的 `ObservableObserveOn` 内部的 `ObserveOnObserver` 是一个 `QueueDisposable` 对象，而 `ObservableFlatMap` 中的 `InnerObserver` 在 `onSubscribe` 时判断到是 `QueueDisposable` 就会调用 `ObserveOnObserver` 的 `requestFusion` 方法，会返回 `ASYNC`。

`Observable.create` 内的 `onNext` 传到 `flatMap`，因为调用了 `observeOn`，所以 `flatMap` 内部发送 `onNext` 要经过调度线程，所以 `ObserveOnObserve` 内还没有从队列取数据往下游（也就是 `ObservableFlatMap` 中的 `InnerObserver` ）发送，`MergeObserver` 的 `onComplete` 就已经执行了。调用 `drain` 然后会调用到 `drainLoop`，不断从队列中获取数据，队列来自于 `ObserveOnObserver` 。等到 `onNext` 执行的时候，在 `InnerObserver` 中 `onNext` 仍然是执行 `drain`，而因为 `drainLoop` 已经开始执行，所以不会再次调用 `drainLoop`。
```
//InnerObserver

public void onNext(U t) {
    if (fusionMode == QueueDisposable.NONE) {
        //正常情况，直接向下游传递数据
        parent.tryEmit(t, this);
    } else {
        //末尾是QueueDisposable，则是不断读队列，直到onComplete
        parent.drain();
    }
}

//MergeObserver

void drain() {
    if (getAndIncrement() == 0) {
        drainLoop();
    }
}
```
所以此时 `drainLoop` 内往下游调用 `onNext` 是在 `flatMap` 的订阅线程，并不会受到末尾的 `observeOn` 影响。而如果在 `observeOn` 后再调用例如 `map`，此时对于 `InnerObserver` 来说，它的上游不再是 `QueueDisposable` ，所以此时 `observeOn` 可以生效。 

所以最终可以总结，如果在 `flatMap` 参数内部返回的  `Observable` 末尾调用了 `observeOn`，那么往下游传递数据时的线程，就取决于 `drain()` 是被 `onNext` 还是 `onComplete` 调用的（只有第一次调用有效），谁先调用，执行的线程就是谁所在的线程。可见，最好不要在 `flatMap` 内使用 `observeOn`，可能RxJava原本就未考虑这种情况。