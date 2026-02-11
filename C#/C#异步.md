## 线程
`System.Threading`命名空间包含了线程相关的类。`Thread` 类型的API和Java类似。

### 锁
* lock
* ReadWriteLock
* ReadWriteLockSlim
* Interlocked CAS类型
* Monitor 可以设置超时时间
* SpinLock 自旋锁
* WaitHandle
* Mutex 是提供跨多个进程同步的类之一
* Semaphore 信号量，可以跨进程
* SemaphoreSlim 轻量级的信号量，用于在单个进程内同步线程。支持异步
* ManualResetEvent 手动重置事件，用于在一个线程完成后通知其他线程继续执行。
* Barrier

### Volatile
C#既有`volatile`关键字，也有`Volatile`类。`volatile` 是“字段修饰符”，编译期就确定；`Volatile` 是“工具类”，运行期想用时再套一层。
```csharp
private static int _stop;

static void Worker()
{
    while (Volatile.Read(ref _stop) == 0)
    {
        // do work
    }
}

static void Main()
{
    var t = new Thread(Worker);
    t.Start();

    Thread.Sleep(1000);
    Volatile.Write(ref _stop, 1);   // 安全发布
    t.Join();
}
```
> 因为`Volatile`相关函数要读写原本的变量，所以要使用`ref`，传递引用，而不是副本

### SynchronizationContext
`SynchronizationContext` 提供了一种机制，用于将任务或回调调度到特定线程或上下文。它抽象了不同线程模型的细节，允许开发者在不同环境中以一致的方式处理线程同步问题。简单来说，它是一个“线程上下文的代理”，可以帮助你将代码执行调度到正确的线程。

```csharp
private void button1_Click(object sender, EventArgs e)
{
    var uiCtx = SynchronizationContext.Current; // 获取当前 UI 线程的上下文
    Task.Run(() =>
    {
        // 线程池线程
    }).ContinueWith(t =>
    {
        // 送回 UI 线程
        uiCtx.Post(_ => button1.Text = "Done", null);
    });
}
```

`await` 语法糖帮你自动包好了：
```csharp
async Task UpdateUIAsync()
{
    // 编译器自动捕获【当前的UI线程同步上下文】
    await Task.Run(() => DoWork()); // 后台线程
    // 编译器自动切回【捕获的上下文线程】执行，所以可以直接修改UI
    UpdateUI();
}
```

> 一般情况，SynchronizationContext.Current 都是 null，开发桌面程序时，`.NET`框架会自动创建并自动绑定到主线程

## Task
主要是 `System.Threading.Tasks` 命名空间，但`System.Threading`命名空间中的一些API，也会用于`Task`，比如`AsyncLocal<T>`

`Parallel.For`、`Parallel.ForEach` 、`Parallel.Invoke` 并行循环，方便执行简单的多个重复任务，适合CPU密集型操作。（占用线程池线程直到任务完成，每个任务独占一个线程）
```c#
public static void ParallelFor()
{
  ParallelLoopResult result = Parallel.For(0, 10, i =>
  {
    Log($"S {i}");
    Task.Delay(10).Wait();
    Log($"E {i}");
  });
  Console.WriteLine($"Is completed: {result.IsCompleted}");
}
```

创建Task的方式：
```c#
// TaskFactory 提供创建和调度Task的支持
var task = Task.Factory.StartNew(()=>{}) // 静态 Task<TResult>.Factory 属性返回默认的 TaskFactory<TResult> 对象

// 还可以调用 TaskFactory 类的构造函数来配置 TaskFactory 类创建的 Task 对象。
var task factory = new TaskFactory(
      cts.Token,
      TaskCreationOptions.PreferFairness,
      TaskContinuationOptions.ExecuteSynchronously,
      new CustomScheduler()); 
factory.StartNew(() => {
    
});

// Factory还提供 ContinueWhenAll 等方法，用于在多个任务完成后执行 continuation 任务。
factory.ContinueWhenAll(new[] { task1, task2 }, tasks => {
    Console.WriteLine("All tasks completed");
});


// Task 构造函数（还有其他重载函数，比如传入 CancellationToken 参数）
var task = new Task(() =>{});
task.Start();

// Task 静态方法 Run 会创建一个新的任务并立即开始执行，返回一个 Task 对象。（也有其他重载函数）
var task = Task.Run(() => {
    
});
```

同步执行
```c#
task.RunSynchronously();
```

`TaskCreationOptions`枚举 指定用于控制创建和执行任务的可选行为的标志。
```c#
None：默认行为
PreferFairness：任务调度器会尽量将任务分配给不同的线程，而不是所有任务都在一个线程上执行。
LongRunning：指定任务是一个长时间运行的任务，会创建新线程，而不是使用线程池中的线程。
AttachedToParent：指定任务是一个子任务，与父任务相关联。
DenyChildAttach：指定任务不允许创建子任务。
HideScheduler：指定任务不使用任务调度器。
RunContinuationsAsynchronously：指定任务完成后，继续任务的执行是异步的。
```

有结果的 Task，Task<TResult>类型
```c#
var task = new Task<string>( () =>
{
    return "";
});
```

可以使用异步函数的构造版本，这样就可以在Task内运行异步函数：
```c#
// 构造函数版本
var task = new Task(async () =>
{
    await Task.Delay(1000);
});

// public Task Run(Func<Task> function)
Task.Run(async () =>
{
    await Task.Delay(1000);        
});


// Task.Factory.StartNew 版本：
```c#
Task.Factory.StartNew(async () =>
{
    await Task.Delay(1000);        
});
```
当使用 `async lambda` 作为参数时：
* 外层 StartNew 返回的 Task 表示启动异步操作的任务
* 内层 Task 是 async lambda 自身返回的异步操作
* 这会形成 Task<Task> 的嵌套结构

> 建议使用`Unwrap()`，或者最好使用`Task.Run（最佳实践）`

**为什么不建议用Task构造函数？**因为Task构造函数的设计初衷是用于同步代码的封装，async/await 时代到来后，`async lambda`会返回`Task<TResult>`类型，使用`new Task(async () => ...)`或者`Task.Factory.StartNew(async () => ...)`返回又会嵌套一层，变成`Task<Task<TResult>>`，需要手动处理`Unwrap()`，容易造成"fire-and-forget"问题（忘记await）
```c#
// 错误示范 1：直接用 Task 构造函数 + async lambda
var task = new Task(async () =>
{
    await Task.Delay(1000);
    Console.WriteLine("内部完成");
    // 假设这里抛异常 throw new Exception("boom");
});

task.Start();

// 外面这样用：
await task;   // ← 这一步几乎立刻就结束了！！
Console.WriteLine("外面以为完成了");
```

> 因为C#的async、await是在Task之后才出现的，兼容原因，所以这里会有多套一层的问题，不像Swift、Kotlin的异步函数返回值可以直接使用类型，而不需要套在Task中

> `Unwrap()` 的作用是把 `Task<Task>` 或 `Task<Task<T>>` “拆开”（unwrap），变成一个普通的 `Task` 或 `Task<T>`，让这个 `Task` 真正代表内层异步操作的最终完成状态，而不是只代表“启动了内层 Task”。


### ContinueWith
task1 执行完后，执行 task2:
```c#
var task2 = task1.ContinueWith(task =>
{
                
});
```

### TaskCompletionSource
`TaskCompletionSource`可以用来将传统异步代码改造为 Task：
```c#
// 场景：从第三方回调中获取结果，包装成 Task
public Task<string> GetDataFromCallbackAsync()
{
    var tcs = new TaskCompletionSource<string>();

    // 模拟第三方回调（比如事件、消息队列回调）
    SomeThirdPartyApi.OnDataReceived += (data) =>
    {
        tcs.TrySetResult(data);           // 成功完成
        // 或 tcs.TrySetException(new Exception("出错了"));
        // 或 tcs.TrySetCanceled();
    };

    return tcs.Task;
}
```

### 结构化并发相关验证
**默认情况下，C#的 Task 之间并不会有父子关系**。

#### 旧版本的父子级关联 AttachedToParent
如果使用 `Task.Factory.StartNew` 配合 `TaskCreationOptions.AttachedToParent` 参数创建 Task，可以让父级 Task 等待子级完成后才完成：
```c#
var parent = Task.Factory.StartNew(() =>
{
    Console.WriteLine("Parent start");

    var child = Task.Factory.StartNew(() =>
    {
        Thread.Sleep(1000);
        Console.WriteLine("Child done");
        // throw new Exception("Child error");  // 会传播到父级
    }, TaskCreationOptions.AttachedToParent);
    Console.WriteLine("Parent done");

    // 父会等待 child
});

await parent; // 自动等 child

// 父级 Wait() 可以获取到子级的异常
// try
// {
//     parent.Wait();
// }
// catch (Exception e)
// {
//     Console.WriteLine(e);
// }

Console.WriteLine("await parent finished");

// 打印结果：
// Parent start
// Parent done
// Child done
// await parent finished
```
父级可以在子级完成后自身才完成，并且会将子级的异常整合为`AggregateException`。如果父任务在子任务之前完成，父任务的状态为 `WaitingForChildrenToComplete`，所有子任务完成，父任务状态变为`RanToCompletion`；不过如果使用`TaskCreationOptions.DetachedFromParent`，则并非如此。

> 现代 Task 基本不使用这个方式

#### 旧版本的父子级关联 ContinueWith、WaitAll
没有 async、await 的时候，可以使用`ContinueWith`、`WaitAll`来等待其他 Task。


#### 现代 Task
现代 task 支持了 `async`、`await` 语法糖关键字，会自动生成状态机。当然，对于结构化并发相关的讨论，重点则是`CancellationToken`，它可以实现**部分**结构化特性。（C# 的 Task 体系本质上仍然是非结构化，需要基于`CancellationToken`手动实现）

##### CancellationToken 基本使用
先看`CancellationToken`的基本使用：
```c#
async Task MainAsync()
{
    // 需要通过 CancellationTokenSource 来得到 CancellationToken
    var cts = new CancellationTokenSource();
    CancellationToken token = cts.Token;

    var task = LongRunningWorkAsync(token);

    // 模拟 1 秒后用户取消
    await Task.Delay(1000);
    cts.Cancel();

    try
    {
        await task;
        Console.WriteLine("任务正常完成");
    }
    catch (OperationCanceledException)
    {
        Console.WriteLine("任务被取消");
    }
}

async Task LongRunningWorkAsync(CancellationToken token)
{
    for (int i = 0; i < 100; i++)
    {
        // 关键：定期检查是否被取消
        token.ThrowIfCancellationRequested();

        Console.WriteLine($"正在处理 {i}...");
        await Task.Delay(200, token);  // 支持取消的 Delay
    }
}

await MainAsync();

// 打印结果：

// 正在处理 0...
// 正在处理 1...
// 正在处理 2...
// 正在处理 3...
// 正在处理 4...
// 任务被取消
```
`CancellationTokenSource`可以控制取消，也可以读取状态，而`CancellationToken`只能用来读取取消状态。

通过共享同一个 `CancellationToken` 或者说`CancellationTokenSource` 可以让 Task 之间产生关联（只能手动协作，并不是自动关联）。也就是说取消父任务的 `CancellationToken`，子任务不会自动取消，除非子任务也监听同一个`CancellationToken`，并且调用相关方法支持协作取消。

`CancellationToken`可以通过`IsCancellationRequested`或者`ThrowIfCancellationRequested()`来主动检测取消，如果想被动接收取消通知可以使用`Register(...)`方法注册取消监听。

##### CreateLinkedTokenSource
`CancellationTokenSource.CreateLinkedTokenSource`的作用是创建一个新的 CancellationTokenSource，它的 Token 会和一个或多个已有的 Token 绑定在一起，只要任何一个源被取消，这个新 Token 就会被取消。用于实现取消传播的层级结构

##### 多个任务共享token
多个任务共享同一个`CancellationToken`，就可以一起取消：
```c#
async Task ProcessMultipleAsync()
{
    using var cts = new CancellationTokenSource();
    var token = cts.Token;

    var tasks = new[]
    {
        ProcessItemAsync("A", token),
        ProcessItemAsync("B", token),
        ProcessItemAsync("C", token),
    };

    // 3秒后取消全部
    _ = Task.Delay(3000).ContinueWith(_ => cts.Cancel());

    await Task.WhenAll(tasks);
}
```

##### 父子取消联动，模拟结构化并发
```c#
async Task ProcessWithFailFastAsync(CancellationToken externalCt = default)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(externalCt);
    var token = cts.Token;

    var pending = new List<Task>
    {
        DoWorkAsync("A", token),
        DoWorkAsync("B", token),
        DoWorkAsync("C", token),
    };

    Exception? firstException = null;

    while (pending.Count > 0)
    {
        var completed = await Task.WhenAny(pending);
        pending.Remove(completed);

        if (completed.IsFaulted)
        {
            firstException ??= completed.Exception?.InnerException ?? completed.Exception;

            // 发现第一个异常 → 立刻取消所有剩余任务
            cts.Cancel();

            // 可以选择 break，也可以继续等待其他任务（视需求）
            // break;
        }
    }

    if (firstException != null)
    {
        throw firstException;
    }
}

async Task DoWorkAsync(string name, CancellationToken token)
{
    for (int i = 0; i < 10; i++)
    {
        if (name == "A" && i == 3)
        {
            // 模拟异常
            throw new Exception();
        }
        token.ThrowIfCancellationRequested();   // 必须定期检查！
        Console.WriteLine($"{name} 正在工作 {i}");
        await Task.Delay(300, token);
    }
    Console.WriteLine($"{name} 完成");
}

await ProcessWithFailFastAsync();
```
任何一个子级抛出异常，父级捕获异常后取消其他子任务。

子任务异常，异常会传播到父任务，但父任务不会自动取消：
* 如果父任务捕获了子任务的异常（通过 try-catch），父任务可以继续执行。
* 如果父任务没有捕获异常，异常会继续向上传播，最终可能导致整个任务链失败。

默认情况下，子任务的异常不会自动影响其他兄弟任务，这是因为 C# 的 Task 模型没有像 Kotlin 协程那样的结构化并发机制，兄弟任务之间的状态是独立的，除非通过显式的代码逻辑或 `CancellationToken` 进行协调。

##### 子级主动取消
其实上面的例子中，子级得到`CancellationToken`并不能主动取消，因为`CancellationToken`只能用来读取状态，要主动取消需要使用`CancellationTokenSource`。所以想在子级主动取消，就需要传递`CancellationTokenSource`参数，而不是`CancellationToken`。

Kotlin 协程通过结构化并发自动传播取消，C# 需要手动实现类似行为。

### async/await机制
在 C# 中，返回类型为 `Task` 或 `Task<T>` 的函数，即使没有标记为 `async`，也可以使用 `await` 进行等待。这是因为 `await` 的工作机制并不依赖于方法是否被标记为 `async`，而是依赖于方法的返回类型是否实现了 `Awaitable` 模式（通常是 `Task` 或 `Task<T>`）。

如果方法只是简单地返回一个 `Task`（例如通过 `Task.FromResult` 或其他方式），就不需要 `async` 关键字，因为 `async` 主要用于处理方法体内的异步操作（例如 `await` 其他任务）。不使用 `async` 可以避免编译器生成状态机（state machine），从而减少性能开销，尤其是在方法逻辑非常简单时。

`await` 是一个 C# 关键字，用于在异步方法（标记为 `async` 的方法）中等待一个异步操作的完成。它操作的对象必须符合 `Awaitable` 模式，即返回的对象需要有一个 `GetAwaiter` 方法（返回一个 `Awaiter` 类型，通常是 `TaskAwaiter` 或 `TaskAwaiter<T>`）。

在 C# 中，`await` 的本质是对一个实现了 `Awaitable` 模式的对象的异步等待操作。它的核心作用是暂停当前方法的执行，等待异步操作完成，并在完成后恢复执行，同时处理结果或异常。以下是 `await` 的本质和底层机制的详细拆解：
1. **检查异步操作的状态**：`await` 检查目标 Task（或其他 Awaitable 对象）的状态。如果任务已经完成（IsCompleted 为 true），直接获取结果并继续执行；否则，暂停当前方法。
2. **暂停和恢复执行**：如果任务未完成，`await` 会将当前方法的执行挂起，并将后续代码（continuation）注册到任务的完成回调中。当任务完成时，恢复执行后续代码。
3. **处理结果或异常**：
  * 如果任务成功完成（IsCompleted），await 返回任务的结果（对于 `Task<T>`）或继续执行（对于 Task）。
  * 如果任务失败（IsFaulted），await 抛出任务中的异常（通常是 AggregateException 的第一个 InnerException）。
  * 如果任务被取消（IsCanceled），await 抛出 TaskCanceledException。

**上下文捕获**：`await` 默认会捕获当前的同步上下文（`SynchronizationContext` 或 `TaskScheduler`），确保任务完成后的继续执行在正确的上下文（例如 UI 线程）中运行。


`async`函数的返回类型只支持`void`、`Task`、`Task<TResult>`、`ValueTask`、`ValueTask<TResult>`以及 `IAsyncEnumerable<T>` 异步流（C# 8.0+），配合 yield return、`IAsyncEnumerator<T>` 异步枚举器（C# 8.0+）。

在C#中异步函数返回`Task<T>`而非直接类型`T`的设计，与Swift的`async/await`（返回直接类型）和Kotlin的`suspend`函数（返回直接类型）有本质区别，并且 `async` 函数，也可以在同步上下文中直接调用（也就在当前线程开始执行），但一般建议调用`Forget()`明确忽略结果。主要原因如下：
* .NET 4.0时代引入Task：C#的异步模型基于Task并行库(TPL)演进，早期通过Task表示异步操作状态
* 渐进式改进：为保持与旧代码兼容，无法像Swift/Kotlin那样完全重构底层机制

总的来说，C# 的`Task`和`async/await`不是同时开发的，所以用法存在一些过渡、新旧混合的感觉

### Task.WhenAll
可以使用 WhenAll、WhenAny、WhenAllAsync、WhenAnyAsync

### ValueTask
当异步操作可能同步完成时，`ValueTask` 是值类型结构体，可以避免创建 `Task` 对象
```csharp
public ValueTask<string> GetDataAsync(int id)
{
    if (_cache.TryGetValue(id, out var data))
        return ValueTask.FromResult(data); // 同步返回
        
    return new ValueTask<string>(FetchFromNetworkAsync(id)); // 异步路径
}
```
> ValueTask 只能被 await 一次

### TaskScheduler
* ThreadPoolTaskScheduler（默认）：将任务调度到线程池。
* SynchronizationContextTaskScheduler：基于特定的 SynchronizationContext 调度任务（例如，调度到 UI 线程）。

* Task.Run：使用默认线程池 
* Task.Factory.StartNew：可指定任务执行线程

### ConfigureAwait
`ConfigureAwait` 是 Task 和 ValueTask 上提供的一个方法，用于控制 await 一个 Task 完成后，代码应该在什么样的“上下文”中继续执行。

默认情况下，await 会捕获当前的 SynchronizationContext（或 TaskScheduler），并在 await 完成后的延续代码中尝试回到捕获的上下文（例如 UI 线程）。`ConfigureAwait(false)` 则会告诉异步操作不要捕获当前的 SynchronizationContext 或 TaskScheduler，而是直接在线程池线程上执行延续代码，从而避免上下文切换的开销。相当于说：“我不需要特定的上下文，我可以在任意线程继续执行。”


`ConfigureAwait(ConfigureAwaitOptions)`提供比 `ConfigureAwait(bool)` 更灵活的选项，用于优化异步代码的性能、行为和资源使用。
ConfigureAwaitOptions 是一个标志枚举（Flags Enum），允许组合多个选项。主要值包括：
* `None`：默认行为，等同于 ConfigureAwait(false)，不捕获当前的 SynchronizationContext 或 TaskScheduler，延续代码会在任意线程池线程上运行。
* `ContinueOnCapturedContext`：显式要求延续代码在捕获的 SynchronizationContext 或 TaskScheduler 上运行。
* `SuppressThrowing`：如果任务失败，延续代码不会抛出异常，而是返回一个带有异常状态的任务。
* `ForceYielding`：强制 await 即使任务已完成，也将其延续代码调度为异步执行（避免同步完成带来的栈深度问题）。

## 异步流
`IAsyncEnumerable<T>` 是异步版本的 `IEnumerable<T>`，表示一个可以异步产生多个元素的序列。
```c#
// 生产端：异步产生数据
async IAsyncEnumerable<int> GenerateNumbersAsync(
    int count,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < count; i++)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(500, ct);   // 模拟异步工作
        yield return i;
    }
}

// 消费端
await foreach (var number in GenerateNumbersAsync(10))
{
    Console.WriteLine($"收到: {number}");
}
```

## Channel
`Channel` 是线程安全的、异步友好的生产者-消费者队列。
* `Channel.CreateBounded<T>`：有限容量，满了会根据策略阻塞生产者 / 拒绝写入
* `Channel.CreateUnbounded<T>`：无限容量，不会阻塞