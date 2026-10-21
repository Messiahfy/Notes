Swift中可以使用如下几种并发编程方式：
* pthread：POSIX线程API，需要通过C交互，一般不使用
* Thread（对应OC的NSThread）：直接启动线程，和Java的Thread用法类似，但功能更简单，比如没有Join()。也较少使用
* Grand Central Dispatch（GCD）：用于管理线程池和任务队列
* NSOperation（基于GCD）
* Async/Await：Swift 5.5引入的新特性，用于简化异步编程

Swift 提供了多种线程锁和并发安全机制：
* 低级锁：NSLock、NSRecursiveLock、DispatchSemaphore、os_unfair_lock、pthread_mutex。
* 现代并发：Actor、@MainActor、DispatchQueue、Sendable、Mutex（可以查看Swift的`Synchronization`文档部分）

https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html

# GCD
GCD的核心是`DispatchQueue`，它是任务的容器，通过它来执行任务。GCD的使用方式也很简单，就是先创建`DispatchQueue`，然后将任务添加到`DispatchQueue`中执行。

## 核心使用方式

1. 创建`DispatchQueue`

创建`DispatchQueue`对象的初始化参数：
* `label`：队列的名称，用于调试。
* `qos`（类型为DispatchQoS）：队列优先级，下文介绍
* `attributes`（类型为数组）：队列的类型，包括（两种属性可以组合使用）：
  * .concurrent：并行队列，允许多个任务同时执行。
  * .initiallyInactive：创建一个不会自动开始执行任务的队列，需要调用 activate() 方法后才能激活队列开始执行任务。
* `autoreleaseFrequency`：控制自动释放池（Autorelease Pool）的创建和释放频率，默认为 workItem。
  * .inherit：继承目标队列的自动释放频率，如果没有指定目标队列(target)，则等同于.workItem
  * .workItem：为每个任务创建独立的自动释放池，任务执行完毕后会自动释放池中的对象
  * .nerver：不创建自动释放池
* `target`（DispatchQueue类型）：目标队列，队列的任务转发到目标队列，继承目标队列的 QoS 和并发性，决定了任务在GCD全局线程池中的线程调度方式。

**使用`DispatchQueue`的重点在于两个维度：`串行/并行`，`同步/异步`**。

2. 创建任务
创建任务有两种常用方式：
```swift
// 1. 创建 DispatchWorkItem
let workItem = DispatchWorkItem() {
    // ......
}
// 加入到队列中
dispatchQueue.async(execute: workItem)


// 2. 直接使用闭包，创建并加入到队列中
dispatchQueue.async {

}
```

### 串行队列
```swift
// 当前在主线程开始执行：

// 默认为串行队列
let serialQueue = DispatchQueue(label: "serialQueue")
serialQueue.sync {
    print("串行 同步1 \(Thread.current)")
}
print("111")
serialQueue.async {
    print("串行 异步1 \(Thread.current)")
}
print("222")
serialQueue.sync {
    print("串行 同步2 \(Thread.current)")
}
print("333")
serialQueue.async {
    print("串行 异步2 \(Thread.current)")
}
print("444")

// 打印结果：
串行 同步1 <_NSMainThread: 0x600002664bc0>{number = 1, name = main}
111
222
串行 异步1 <NSThread: 0x600002624640>{number = 2, name = (null)}
串行 同步2 <_NSMainThread: 0x600002664bc0>{number = 1, name = main}
333
444
串行 异步2 <NSThread: 0x600002624640>{number = 2, name = (null)}
```
由于是串行队列，无论同步异步，都会按照顺序执行。异步是相对于当前线程，任务之间的顺序并不受影响，必然为串行顺序执行。GCD 通常为每个串行队列分配一个线程（可能复用线程池中的线程）。

> sync会阻塞当前线程，正常情况不应该在主线程调用

### 并行队列
```swift
// 并行队列
let concurrentQueue = DispatchQueue(label: "concurrentQueue", attributes: .concurrent)

concurrentQueue.async {
    print("并行 异步任务1开始 - \(Thread.current)")
    Thread.sleep(forTimeInterval: 2)  // 模拟耗时操作
    print("并行 异步任务1完成")
}
concurrentQueue.async {
    print("并行 异步任务2开始 - \(Thread.current)")
    Thread.sleep(forTimeInterval: 2)  // 模拟耗时操作
    print("并行 异步任务2完成")
}
print("111")
        
concurrentQueue.sync {
    print("并行 同步任务1 - \(Thread.current)")
    Thread.sleep(forTimeInterval: 2)
}
print("222")

concurrentQueue.async {
    print("并行 异步任务3开始 - \(Thread.current)")
    Thread.sleep(forTimeInterval: 1)
    print("并行 异步任务3完成")
}
print("333")
        
concurrentQueue.sync {
    print("并行 同步任务2 - \(Thread.current)")
}
print("444")

// 打印结果：
111
并行 异步任务1开始 - <NSThread: 0x600003c07a80>{number = 2, name = (null)}
并行 同步任务1 - <_NSMainThread: 0x600003c4cb40>{number = 1, name = main}
并行 异步任务2开始 - <NSThread: 0x600003c6e540>{number = 3, name = (null)}
222
333
并行 同步任务2 - <_NSMainThread: 0x600003c4cb40>{number = 1, name = main}
并行 异步任务3开始 - <NSThread: 0x600003c5c000>{number = 4, name = (null)}
444
并行 异步任务1完成
并行 异步任务2完成
并行 异步任务3完成
```
在并行队列中，任务之间的执行顺序是不确定的，因为它们可能在不同的线程上执行。并行队列使用同步任务依然会阻塞当前线程，并且为了性能优化同步任务一般会在当前线程执行，有一个例外是提交到主调度队列的任务始终在主线程上运行。

> 这里演示并行队列时，同步任务在当前线程执行，并且阻塞了当前线程，导致它之后的代码（另外的同步和异步任务）都要等这个同步任务执行完毕才能执行，其实会有一点误导（好像同步任务会导致串行），实际上同步任务并不会影响任务并行，只是类似Java Future的get()方法，如果是在其他线程调用sync，那么就阻塞其他线程，则其他线程的同步任务和当前线程添加的异步任务就可以并行执行了。

还有例如`asyncAfter`用于延迟执行或者其他功能的方法，可以使用时查看文档。

## 内置的队列
GCD 提供了一些内置的队列，用于特定的任务：
* DispatchQueue.main：主队列，用于更新UI。
* DispatchQueue.global()：全局队列，用于执行后台任务。

## 队列优先级（QoS）
GCD 是一个高效的任务调度系统，它使用一个全局线程池来执行任务，而不是为每个队列创建独立的线程池，所有非主队列的队列（包括自定义队列和全局队列）共享 GCD 的全局线程池。全局线程池由 GCD 管理，根据系统资源（如 CPU 核心数、负载）和任务的 QoS（服务质量）动态分配线程。

GCD 使用服务质量（Quality of Service, QoS）控制任务优先级：
* .userInteractive：高优先级，适合 UI 响应。
* .userInitiated：用户触发的任务，如点击按钮。
* .default：默认优先级，通用任务。
* .utility：耗时任务，如文件操作。
* .background：低优先级，后台任务。

创建`DispatchQueue`时和调用队列的`async`方法时，都可以指定 QoS。

## DispatchWorkItemFlags
`async` 和 `sync` 方法还可以设置flags，它是`DispatchWorkItemFlags`类型：
* `.barrier`：用于同步队列，确保在该任务之前的任务完成后再执行，后面的任务等待该任务完成。
* `.detached`：以分离的方式运行任务，任务不继承当前队列的上下文（例如队列、QoS 或其他属性）。
* `.assignCurrentContext`：强制任务绑定到当前执行的队列和上下文（包括队列的目标队列和 QoS）
* `.noQoS`：不使用 QoS。
* `.inheritQoS`：继承父队列的 QoS。
* `.enforceQoS`：强制使用 QoS。

> `DispatchWorkItemFlags`是`OptionSet`类型，所以可以组合使用，比如`[.detached, .barrier]`

### barrier
```swift
// 并行队列
let concurrentQueue = DispatchQueue(label: "concurrentDemo", attributes: .concurrent)

var data = 0

// 添加3个异步读取任务到并行队列中
for _ in 0..<3 {
    concurrentQueue.async {
        let readValue = data
        print("写入前读取 \(readValue) on thread \(Thread.current)")
    }
}

// 添加1个写入任务到并行队列中，并且使用barrier标志
concurrentQueue.async(flags: .barrier) {
    data += 1
    print("写入 \(data) on thread \(Thread.current)")
}

// 添加3个异步读取任务到并行队列中
for _ in 0..<3 {
    concurrentQueue.async {
        let readValue = data
        print("写入后读取 \(readValue) on thread \(Thread.current)")
    }
}

打印结果：
写入前读取 0 on thread <NSThread: 0x600003554480>{number = 2, name = (null)}
写入前读取 0 on thread <NSThread: 0x600003530080>{number = 3, name = (null)}
写入前读取 0 on thread <NSThread: 0x60000352b740>{number = 4, name = (null)}
写入 1 on thread <NSThread: 0x60000352b740>{number = 4, name = (null)}
写入后读取 1 on thread <NSThread: 0x60000352b740>{number = 4, name = (null)}
写入后读取 1 on thread <NSThread: 0x60000352b740>{number = 4, name = (null)}
写入后读取 1 on thread <NSThread: 0x600003530080>{number = 3, name = (null)}
```
写入任务使用了`barrier`标志，它会确保在添加该任务之前添加的任务完成后再执行，之后添加的任务等待该任务完成后再执行。

### detached、assignCurrentContext、noQoS、inheritQoS、enforceQoS
官方文档对于这几个标志的解释都比较模糊，这里直接测试效果对比一下：
```swift
class ViewController: UIViewController {

    override func viewDidLoad() {
        testQoS(fromQueue: .global(qos: .userInteractive), toQueue: .global(), label: "test1")
    
        testQoS(fromQueue: .main, toQueue: .global(qos: .background), label: "test2")
    }

    // 打印当前 QoS
    func printCurrentQoS(prefix: String) {
        print("\(prefix): QoS = \(DispatchQoS.QoSClass(rawValue: qos_class_self()))")
    }

    // 测试函数：在指定 QoS 的队列中提交任务
    func testQoS(fromQueue: DispatchQueue, toQueue: DispatchQueue, label: String) {
        fromQueue.async {
            
            self.printCurrentQoS(prefix: "current")
            
            
            // 1. 默认 flags
            toQueue.async {
                self.printCurrentQoS(prefix: "\(label) - Default flags")
            }
            
            // 2. assignCurrentContext
            toQueue.async(flags: .assignCurrentContext) {
                self.printCurrentQoS(prefix: "\(label) - assignCurrentContext")
            }
            
            // 3. noQoS
            toQueue.async(flags: .noQoS) {
                self.printCurrentQoS(prefix: "\(label) - noQoS")
            }
            
            
            // 4. assignCurrentContext
            toQueue.async(flags: .inheritQoS) {
                self.printCurrentQoS(prefix: "\(label) - inheritQoS")
            }
            
            // 5. assignCurrentContext
            toQueue.async(flags: .enforceQoS) {
                self.printCurrentQoS(prefix: "\(label) - enforceQoS")
            }
            
            // 6. detached
            toQueue.async(flags: .detached) {
                self.printCurrentQoS(prefix: "\(label) - detached")
            }
        }
    }
}

打印结果：
current: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInitiated)
test1 - Default flags: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInitiated)
test1 - assignCurrentContext: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInitiated)
test1 - inheritQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInitiated)
test1 - enforceQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInitiated)
test1 - detached: QoS = Optional(Dispatch.DispatchQoS.QoSClass.default)
test1 - noQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.default)

current: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInteractive)
test2 - enforceQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInitiated)
test2 - Default flags: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test2 - assignCurrentContext: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test2 - noQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test2 - inheritQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test2 - detached: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
```
测试函数使用了嵌套的任务队列来测试各种标志的影响，`test1`的情况是内层队列未指定QoS，只有`detached`和`noQoS`，其他都受到外层队列的QoS影响；`test2`的情况则是内层队列指定了QoS，此时只有`enforceQoS`会使用一个不低于当前任务的QoS属性，其他都直接使用了内层队列设置的QoS属性。

除了队列可以设置`QoS`属性，任务也可以设置`QoS`属性，在使用了`enforceQoS`标志，会强制使用设置的QoS属性，而不受队列影响：
```swift
func testQoS(fromQueue: DispatchQueue, toQueue: DispatchQueue, label: String) {
    fromQueue.async {
            
        self.printCurrentQoS(prefix: "current")
            
            
        // 1. 默认 flags
        toQueue.async(qos: .userInteractive) {
            self.printCurrentQoS(prefix: "\(label) - Default flags")
        }
            
        // 2. assignCurrentContext
        toQueue.async(qos: .userInteractive, flags: .assignCurrentContext) {
            self.printCurrentQoS(prefix: "\(label) - assignCurrentContext")
        }
            
        // 3. noQoS
        toQueue.async(qos: .userInteractive, flags: .noQoS) {
            self.printCurrentQoS(prefix: "\(label) - noQoS")
        }
            
            
        // 4. inheritQoS
        toQueue.async(qos: .userInteractive, flags: .inheritQoS) {
            self.printCurrentQoS(prefix: "\(label) - inheritQoS")
        }
            
        // 5. enforceQoS
        toQueue.async(qos: .userInteractive, flags: .enforceQoS) {
            self.printCurrentQoS(prefix: "\(label) - enforceQoS")
        }
            
            
        // 6. detached
        toQueue.async(qos: .userInteractive, flags: .detached) {
            self.printCurrentQoS(prefix: "\(label) - detached")
        }
    }
}

testQoS(fromQueue: .global(qos: .userInitiated), toQueue: .global(qos: .background), label: "test3")

打印结果：
current: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInitiated)
test3 - enforceQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.userInteractive)
test3 - noQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test3 - Default flags: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test3 - detached: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test3 - inheritQoS: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
test3 - assignCurrentContext: QoS = Optional(Dispatch.DispatchQoS.QoSClass.background)
```
`enforceQoS`强制使用了自己设置的`userInteractive`。但假如`toQueue`为`.global(qos: .utility)`，而`enforceQoS`的任务使用`.background`，那么结果为`.utility`，因为`enforceQoS`设置的任务QoS属性不会低于当前队列的QoS属性。

## DispatchGroup
DispatchGroup 用于协调一组任务，等待所有任务完成后再执行后续逻辑。

核心方法：
* `notify`和`wait`：当组内所有任务都完成后，可以通过`notify`方法执行一个回调；而`wait`方法作用类似，只不过是同步等待（需谨慎使用，以免阻塞主线程）。
* `enter`和`leave`：当leave调用的次数和enter调用的次数一样，则组内所有任务完成。


比如并行执行多个网络请求的场景，使用方式如下：
```swift
let group = DispatchGroup()
let queue = DispatchQueue.global(qos: .userInitiated)

// 模拟多个网络请求
for i in 1...3 {
    group.enter()
    queue.async {
        print("开始请求 \(i)")
        // 模拟异步网络请求
        DispatchQueue.global().asyncAfter(deadline: .now() + .seconds(i)) {
            print("请求 \(i) 完成")
            group.leave()
        }
    }
}

// 所有任务完成后执行
group.notify(queue: .main) {
    print("所有网络请求完成，更新 UI")
}
```
`group.enter()`立即调用了3次，然后等待异步任务中调用3次`group.leave()`，表示组内所有任务完成，则会收到`notify`回调。

如果使用`wait`方法，则会阻塞当前线程，直到所有任务完成：
```
// 省略相同代码

group.wait() 
print("所有网络请求完成，更新 UI")
```

## DispatchSemaphore 信号量
DispatchSemaphore 控制并发访问，限制同时运行的任务数量。
```swift
let semaphore = DispatchSemaphore(value: 2) // 允许 2 个并发任务
let queue = DispatchQueue.global()

for i in 0..<5 {
    queue.async {
        semaphore.wait() // 获取信号量
        print("Task \(i) started")
        Thread.sleep(forTimeInterval: 1)
        print("Task \(i) completed")
        semaphore.signal() // 释放信号量
    }
}

打印结果：
Task 0 started
Task 1 started
Task 0 completed
Task 1 completed
Task 2 started
Task 3 started
Task 2 completed
Task 3 completed
Task 4 started
Task 4 completed
```
通过DispatchSemaphore，可以实现最多只有2个任务在运行。

## DispatchSource
`DispatchSource`是一个事件源，负责监控特定类型的事件，通常与`DispatchQueue`结合使用，将事件处理任务分派到指定的队列上执行。

DispatchSource 支持以下几种事件类型：
1. Timer：定时器事件，用于定期或一次性执行任务。
2. Signal：监控 Unix 信号（如 SIGINT、SIGTERM）。
3. File System：监控文件或目录的变化（如文件读写、删除、移动）。
4. Process：监控进程状态（如进程退出、fork、exec）。
5. Read/Write：监控文件描述符的读写事件。
6. Mach Port：监控 Mach 端口事件（主要用于 macOS 底层通信）。
7. User Data：自定义事件，用于手动触发事件（如通过 mergeData）。

比如**文件系统监控**：
```swift
let filePath = "/path/to/file.txt"
let descriptor = open(filePath, O_EVTONLY) // 打开文件描述符
guard descriptor >= 0 else { return }
let source = DispatchSource.makeFileSystemObjectSource(
    fileDescriptor: descriptor,
    eventMask: .write, // 监控写入事件
    queue: .main
)
source.setEventHandler {
    print("文件 \(filePath) 被修改")
}
source.setCancelHandler {
    close(descriptor)
    print("文件监控已取消")
}
source.activate()

// 模拟程序运行一段时间后取消
DispatchQueue.global().asyncAfter(deadline: .now() + .seconds(10)) {
    source.cancel()
}
```

# OperationQueue
https://developer.apple.com/documentation/foundation/operation/

使用主要流程：
1. 创建队列OperationQueue
2. 创建任务
3. 任务加入队列，会自动被执行

通过`maxConcurrentOperationCount`控制串行或并行

```swift
// 1. 创建OperationQueue（相当于GCD的队列）
let operationQueue = OperationQueue()

// 2. 创建NSOperation子类或使用BlockOperation

// 方式一：使用自定义Operation
class CustomOperation: Operation {
    override func main() {
        guard !isCancelled else { return }
        print("执行自定义操作")
    }
}
let customOp = CustomOperation()
operationQueue.maxConcurrentOperationCount = 2 // 设置最大并发数

// 方式二：使用BlockOperation（类似GCD的block）
let blockOp = BlockOperation {
    print("执行块操作")
}

// 方式三：直接调用 operationQueue.addOperation(block)

// 3. 添加依赖，设置任务的执行顺序（重要特性）
blockOp.addDependency(customOp) // blockOp会在customOp完成后执行

// 4. 添加到队列
operationQueue.addOperation(customOp)
operationQueue.addOperation(blockOp)

// 5. 管理队列（可取消尚未执行的任务）
// operationQueue.cancelAllOperations() // 取消全部
// customOp.cancel() // 取消单个
```
以上代码示例就是`OperationQueue`的基本使用流程。

## 执行块
还可以向`Operation`添加执行块，用于向一个现有的`BlockOperation`实例动态添加额外的执行闭包（execution block）。这个方法允许你在创建 BlockOperation 后，附加多个并行执行的任务块，并发执行（但受限于 maxConcurrentOperationCount）
```
operation.addExecutionBlock {
    // 执行任务
}
```
开发者可以选择创建多个`BlockOperation`，也可以只创建一个`BlockOperation`并多次调用`addExecutionBlock`来添加不同的任务块，根据情况选择。比如只创建一个`BlockOperation`并且多次调用`addExecutionBlock`，`BlockOperation`会在它内部的所有执行块执行完后`isFinished`属性才为true，状态管理会更方便。

## 异步配置
* `operation.isAsynchronous`：针对单个 Operation 的属性，决定该操作的执行方式（同步或异步）
* `maxConcurrentOperationCount`：针对整个 OperationQueue 的属性，控制队列中同时运行的操作数量。

将operation添加到OperationQueue时，队列将忽略`isAsynchronous`属性的值，并会调用`start()`方法。因此，如果通过将operation添加到OperationQueue来执行operation，则没有必要使用`isAsynchronous`属性。

> 尽管OperationQueue是运行Operation的最便捷方式，但也可以在没有OperationQueue的情况下运行Operation。但是，如果选择手动运行Operation，则应在代码中采取一些预防措施。特别是，该Operation必须准备好运行，并且您必须始终使用其 start 方法启动它。异步操作需手动管理状态，毕竟异步情况下什么时候执行完成是我们的异步代码决定的，Operation无法知道，所以我们需要手动管理状态。

## 线程队列
* OperationQueue.main：主线程队列
* OperationQueue.current：当前线程队列

OperationQueue 不同于 GCD 的队列，它只有两种队列：主队列和其他队列，`OperationQueue.main`就是主线程队列，而直接使用`OperationQueue()`构造函数创建的就是其他队列（非主线程队列）。其他队列用来实现串行和并发的功能，只要操作对象添加到队列，就会自动调用操作对象的`start()`方法。

## 优先级
* `operation.queuePriority` 控制优先级，表示Operation在OperationQueue中的调度优先级，决定操作在队列中的执行顺序。但仅影响操作在 OperationQueue中的调度顺序（即哪些操作先被选中执行），不直接影响底层线程优先级或资源分配。
* `qualityOfService` 是系统级的优先级，影响Operation的资源分配和线程优先级，适用于优化性能和响应性。

> queuePriority 是队列级的优先级，仅决定操作在队列中的调度顺序，不直接影响系统资源。高 queuePriority（如 .veryHigh）仅确保操作更早被队列选中执行，但执行速度取决于 qualityOfService 和系统资源。

operation.qualityOfService 优先级，类型为 QualityOfService 枚举，包含以下值：
* .userInteractive：最高优先级，适合需要立即响应的 UI 相关任务（例如动画）。
* .userInitiated：高优先级，适合用户触发的任务（例如加载数据）。
* .default：默认优先级，系统自动选择（通常中等优先级）。
* .utility：中等偏低的优先级，适合耗时但不紧急的任务（例如下载）。
* .background：最低优先级，适合后台任务（例如日志同步）。


## 其他操作
* operationQueue.isSuspended 控制队列是否暂停，暂停后不会执行队列中的任务，但是可以添加新任务
* operation.completionBlock 任务完成后执行的闭包
* operationQueue.addOperations 可以设置 `waitUntilFinished`，如果为true，会阻塞当前线程，直到所有添加的操作完成
* operationQueue.waitUntilAllOperationsAreFinished() 阻塞当前线程，直到接收方的所有排队和正在执行的作都完成执行。

向 OperationQueue 添加 Operation 后，队列会自动开始执行（前提是 operationQueue.isSuspended = false 且 operation.isReady = true）。如果operation有依赖（operation.dependencies 不为空）， isReady 只有在所有依赖操作的 isFinished = true 时才为 true

# Async/Await
Swift 5.5 引入了 Async/Await 特性，也就是 Swift 协程，这是更现代化的异步编程方式，Swift协程和Kotlin协程有很多相似之处。

## 基本使用
1. 编写一个`async`函数

模拟网络请求延迟1秒后返回字符串结果，将回调函数包装为了一个`async`函数，这里的重点是调用了`withCheckedContinuation`函数，它和Kotlin中的`suspendCoroutine`函数类似，都是提供了`continuation`。
```swift
func networkRequest(key: Int) async -> String {
    await withCheckedContinuation { continuation in
        DispatchQueue.global().asyncAfter(deadline: .now() + 1.0) {
            continuation.resume(returning: "结果\(key)")
        }
    }
}
```

`withCheckedContinuation`函数声明如下：
```swift
public func withCheckedContinuation<T>(
    isolation: isolated (any Actor)? = #isolation,
    function: String = #function,
    _ body: (CheckedContinuation<T, Never>) -> Void
) async -> sending T
```

`CheckedContinuation`结构体的声明如下：
```swift
public struct CheckedContinuation<T, E> : Sendable where E : Error {

    public init(continuation: UnsafeContinuation<T, E>, function: String = #function)
    
    public func resume(returning value: sending T)

    public func resume(throwing error: E)
}
```

2. 通过`Task`执行`async`函数

和Kotlin协程一样，异步函数需要在另一个异步函数中调用，但总归有一个初始调用点，Kotlin的挂起函数初始调用点是协程作用域，而Swift的初始调用点则是`Task`.

`Task`的常用构造函数如下，可以传入一个异步闭包：
```swift
public init(priority: TaskPriority? = nil, operation: sending @escaping @isolated(any) () async -> Success)

public init(priority: TaskPriority? = nil, operation: sending @escaping @isolated(any) () async throws -> Success)
```

所以可以这样调用异步函数：
```swift
Task {
    let result = await networkRequest(1)
    print("结果为 \(result)")
}
```

3. 获取`Task`的结果（可选）
`Task`的闭包返回值会作为`Task`的返回值，并且由于`Task`是异步执行的，所以它的结果也是异步的。
```swift
public var value: Success { get async }
```

所以要获取结果，也要在另一个异步函数或者`Task`中执行：
```swift
// 一个实际不太会这样用的例子，只是展示用法
Task {
    let result = await Task {
        await networkRequest(1)
    }.value
    print("结果 \(result)")
}
```

## TaskGroup
无论是`Task.init`还是`Task.detached`，都是非结构化的，但可以使用`TaskGroup`来管理它们。

### 基本使用
1. 创建TaskGroup

`withTaskGroup`是一个`async`函数，它会在 TaskGroup 内所有的子 Task 执行完之后再返回。
```swift
@inlinable public func withTaskGroup<ChildTaskResult, GroupResult>(
    of childTaskResultType: ChildTaskResult.Type = ChildTaskResult.self, 
    returning returnType: GroupResult.Type = GroupResult.self, 
    isolation: isolated (any Actor)? = #isolation, 
    body: (inout TaskGroup<ChildTaskResult>) async -> GroupResult
) async -> GroupResult where ChildTaskResult : Sendable
```
前两个是泛型参数：
* `ChildTaskResult`：表示这个 TaskGroup 内创建的 Task 的结果类型
* `GroupResult`：TaskGroup 的返回结果类型，也是参数 body 的返回值类型
* `body`：一个异步闭包，它接收一个 TaskGroup 类型的参数，用于添加子 Task

> `withThrowingTaskGroup`对应可以抛出异常的情况、`withDiscardingTaskGroup`对应不需要每个子任务返回值，只关心是否完成的情况。

2. 添加子任务并且获取结果
`TaskGroup`提供了`addTask`方法用于添加子任务，并且可以通过`for await`遍历获取结果。
```swift
Task {
    await withTaskGroup(of: String.self) { group in
        // 添加任务到组
        for id in 1...3 {
            group.addTask {
                await self.networkRequest(key: id)
            }
        }
        
        for await result in group {
            print("打印 TaskGroup 中的结果 \(result)")
        }
    }
}
```

> 还有一个`addTaskUnlessCancelled`方法，它会在`TaskGroup`未被取消的状态下添加任务，如果已取消则不会添加。

`TaskGroup`是`AsyncSequence`的子类，所以可以通过`for await`来遍历结果，并且还有`filter`、`map`、`next`等遍历方法可以使用。
```swift
extension TaskGroup : AsyncSequence {
    // ...
}
```

> AsyncSequence 与 Sequence 的不同之处在于它的迭代器相关遍历函数是异步函数

### 其他方法
`TaskGroup`还提供了一些常用方法：
1. `waitForAll()`：等待所有子任务执行完成
2. `cancelAll()`：取消全部子任务

## 非结构化并发
通过`Task`的构造器或者`detach`函数创建的`Task`实例都是顶级的，多个`Task`之间不会形成任务树，不会自动传递取消和异常，也就是非结构化的。
```swift
// 🚫并不是结构化并发
Task {
    Task {

    }

    Task {

    }
}
```
这种嵌套结构的Task看起来像可以形成任务树，但实际并不会，并不是结构化并发。但`Task`可以继承外层`Task`的优先级、task local values、actor。但如果使用`Task.detached`则不会继承。

## 结构化并发
形成结构化任务树的方式只有：`async let`和`TaskGroup`。

1. `async let`
前面展示调用都是直接对`async`函数使用了`await`，这就相当于挂起等待顺序执行了，这和Kotlin中调用挂起函数的默认效果一样。如果想要并发执行，可以使用`async let`和`await`结合`数组`或者`元组`等待多个异步任务并发执行：
```swift
Task {
    do {
        async let result1 = networkRequest(key: 1)
        async let result2 = networkRequest(key: 2)
        // 使用元组
        let (a, b) = await (result1, result2)
        print("打印 \(a) \(b)")

        // 或者使用数组
        // let results = await [user1, user2, user3]

    } catch {
        print("发生异常")
    }
}
```
通过`async let`实现了两个`networkRequest`并发执行。


**注意**，下面这个形式看起来可以并发，其实是顺序执行的，必须先使用`async let`
```
let (a, b) = await (networkRequest(key: 1), networkRequest(key: 2)) // 🚫 顺序执行，不会并发
```

2. `TaskGroup`
前文介绍`TaskGroup`时已经介绍了它的基本用法，下面重点以取消和异常的场景来着重理解它们在结构化并发场景的作用。

## 取消和异常
`Task`的取消只是一个状态标记，它不会强制终`Task`的代码执行流程，而是和Kotlin协程类似，取消是协作式的，需要添加额外的代码来配合响应取消，从而实现完善的取消逻辑。调用`Task.cancel()`方法，会产生以下影响：
1. 修改`Task.isCancelled`状态标记
2. 触发 CancellationHandler 回调
3. 取消结构化并发相关联的任务

###  `Task.isCancelled`

`Task.isCancelled`是一个只读属性，用于检查当前任务是否已被取消。用法示例：
```swift
let task = Task {
    for key in 1...3 {
        // 每次循环时，检测取消状态，如果已经取消就结束循环
        guard !Task.isCancelled else { return }

        let result = await networkRequest(key: key)
        print("结果 \(result)")
    }
}

// 在另一个Task中等待1秒，取消task
Task {
    try await Task.sleep(nanoseconds: 1_000_000_000)
    task.cancel()
}
```

### `CancellationError`异常和`Task.checkCancellation()`
开发者可以在异步函数中主动抛出`CancellationError`，来直接结束若干层嵌套异步函数。

或者可以使用`Task.checkCancellation()`检测当前Task是否已取消，如果当前`Task.isCancelled`为true，就会自动抛出`CancellationError`。

> 实际验证感觉抛出`CancellationError`或者抛出其他Error对于结构化并发的影响是一样的，只是一个语义方面的区别。

### `withTaskCancellationHandler`注册取消回调
使用`Task.isCancelled`或者`Task.checkCancellation()`都是主动检测是否已经取消，但有些场景更适合注册回调被动接收取消通知。

比如这个异步函数，使用`withTaskCancellationHandler`注册取消回调，收到取消回调时会取消gcd的异步任务，并且使用`continuation.resume`传递回一个`CancellationError()`。
```swift
func cancelableThrowNetworkRequest(key: Int) async throws -> String {
    var block: DispatchWorkItem? = nil
    var continuation: CheckedContinuation<String, Error>? = nil
     try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { cont in
            continuation = cont  // 存储 continuation 以便在 onCancel 中使用
            block = DispatchWorkItem {
                print("networkRequest \(key) 结束")
                continuation?.resume(returning: "结果\(key)")
            }
            DispatchQueue.global().asyncAfter(deadline: .now() + 1.0, execute: block!)
        }
    } onCancel: {
        block?.cancel()
        continuation?.resume(throwing: CancellationError())
    }
}
```

使用`withTaskCancellationHandler`有个需要注意的地方：**`onCancel`取消回调的情况，也需要调用`continuation.resume`，否则异步函数会一直挂起不返回（Kotlin协程中取消回调情况可以不用手动resume，但Swift需要手动处理）。**

或者可以像这个例子，虽然`onCancel`中没有调用`continuation.resume`，但`onCancel`中调用`urlSessionTask?.cancel()`会触发回调error，在`URLSession.shared.dataTask`的回调中就会在取消的情况下执行`continuation.resume(throwing: error)`。
```swift
func download(url: URL) async throws -> Data? {
  var urlSessionTask: URLSessionTask?

  return try withTaskCancellationHandler {
    return try await withUnsafeThrowingContinuation { continuation in
      urlSessionTask = URLSession.shared.dataTask(with: url) { data, _, error in
        if let error = error {
          // Ideally translate NSURLErrorCancelled to CancellationError here
          continuation.resume(throwing: error)
        } else {
          continuation.resume(returning: data)
        }
      }
      urlSessionTask?.resume()
    }
  } onCancel: {
    urlSessionTask?.cancel() // runs immediately when cancelled
  }
}
```


> 可以参考官方实现的支持取消的`Task.sleep(nanoseconds duration: UInt64)`异步函数，https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/TaskSleep.swift#L240

### 异常的影响
在异步函数中抛出异常，如果没有被catch住，异常抛出到了顶级Task，则这个顶级Task会取消它内部的全部child Task，`async let`和`TaskGroup`这两种结构化并发的场景的异常处理都是如此。

来看一个使用`async let`结构化并发中抛出异常的例子：
```swift
// 异步函数，1秒后抛出异常
func asyncThrowError() async throws -> String {
    print("asyncThrowError 开始")
    return try await withCheckedThrowingContinuation { continuation in
        DispatchQueue.global().asyncAfter(deadline: .now() + 1.0) {
            print("asyncThrowError 抛出取消异常")
            continuation.resume(throwing: CancellationError())
        }
    }
}

// 异步函数，2秒后返回结果
func cancelableAsyncRequest() async throws -> String {
    print("cancelableAsyncRequest 开始")
    var block: DispatchWorkItem? = nil
    var continuation: CheckedContinuation<String, Error>? = nil

    return try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { cont in
            continuation = cont

            block = DispatchWorkItem {
                print("cancelableAsyncRequest 正常结束")
                continuation?.resume(returning: "结果")
            }
            DispatchQueue.global().asyncAfter(deadline: .now() + 2.0, execute: block!)
        }
    } onCancel: {
        print("收到取消")

        block?.cancel()
        continuation?.resume(throwing: CancellationError())
    }
}

let task = Task {
    // 并发执行
    async let x = asyncThrowError()
    async let y = cancelableAsyncRequest()
    
    try await (x, y)
}

Task {
    print("task结果： \(await task.result)   isCancelled: \(task.isCancelled)")
}

打印结果：
asyncThrowError 开始
cancelableAsyncRequest 开始
asyncThrowError 抛出取消异常
收到取消
task结果： failure(Swift.CancellationError())   isCancelled: false
```
`asyncThrowError()`执行1秒后抛出异常，顶级Task收到异常后会取消全部子任务，所以`cancelableAsyncRequest()`这个子任务会收到取消回调，从而提前结束。

不过这里需要注意的是，`async let`并发场景，`await`的顺序对异常处理有影响：**Task会收到正在await的子任务抛出的异常，然后取消全部子任务（其他还没有await的子任务）**，所以如果把上面例子中`await (x, y)`改为`await (y, x)`，则`cancelableAsyncRequest()`不会收到取消回调，也就不会提前结束。不过最终Task的result还是会收到异常，但无法及时取消子任务。
```swift
let task = Task {
    async let x = asyncThrowError()
    async let y = cancelableAsyncRequest()
    
    // 修改顺序，先await cancelableAsyncRequest，然后再await asyncThrowError
    try await (y, x)
}

Task {
    print("task结果： \(await task.result)   isCancelled: \(task.isCancelled)")
}

打印结果：
asyncThrowError 开始
cancelableAsyncRequest 开始
asyncThrowError 抛出取消异常
cancelableAsyncRequest 正常结束
task结果： failure(Swift.CancellationError())   false
```

> 这里有这个await顺序问题的讨论：https://forums.swift.org/t/async-let-cancellation-bug-confused/51384 

**如果想要避免`await`顺序对异常处理流程的影响，可以使用`TaskGroup`来并发执行，`TaskGroup`收到子任务的异常会自动取消所有子任务，不受`await`顺序影响**

TaskGroup方式的代码：
```swift
let task = Task {
    try await withThrowingTaskGroup(of: String.self) { group in
        group.addTask {
            try await self.cancelableAsyncRequest()
        }

        group.addTask {
            try await self.asyncThrowError()
        }

        // 需要使用 for await 或者其他遍历、await 的方式来等待结果，否则不能收到子任务抛出的异常
        for try await t in group {
            print("t: \(t)")
        }
    }
}

Task {
    print("task结果： \(await task.result)   \( task.isCancelled)")
}

打印结果：
cancelableAsyncRequest 开始
asyncThrowError 开始
asyncThrowError 抛出取消异常
收到取消
task结果： failure(Swift.CancellationError())   false
```

**抛出异常的总结如下：**
1. 抛出异常到顶级 Task，会自动取消所有 Child Task，所有 Child Task 的`isCancelled`状态都会变成true，也会收到取消回调
2. 接收到异常的顶级 Task 自身的`isCancelled`状态不会变为true（感觉有点怪），除非调用它的`cancel()`方法
3. 只有当子任务的错误真正传递到父任务（通过await）时，父任务才取消全部子任务。比如通过do-catch捕获异常导致异常没有传递到Task，也无法触发取消
4. `TaskGroup`可以避免`async let`的异常处理受到`await`顺序的影响

> Kotlin抛出`CancellationException`异常，如果没有被catch，协程会被取消，`Job.isCancelled`状态变为true，整个结构化并发的任务树都会取消，Swift抛出任何异常都会导致Task内的结构化并发任务树取消，child task的`isCancelled`状态变为true，但是顶级的`Task.isCancelled`状态不会改变。

## 任务树
结构化并发形成的任务树，可以通过`withUnsafeCurrentTask`函数打印验证：
```swift
withUnsafeCurrentTask { task in
    // 打印当前 Task 的 hashValue
    print("CurrentTask \(task?.hashValue)")
}
```

以如下代码为例，我们可以验证确认有5个 Task：
```swift
func asyncRequest() async throws -> String {
    // ......
}

Task { // 顶级Task

    async let a = asyncRequest() // 在 asyncRequest 函数内为 child task 1

    async let b =  withThrowingTaskGroup(of: String.self) { group in
        // child task 2

        group.addTask {
            // child task 3
            try await asyncRequest()
        }

        group.addTask {
            // child task 4
            try await asyncRequest()
        }
        for try await t in group {
            // ...
        }
    }
    try await (b, a)
}
```
顶级Task 
├── child task 1
└── child task 2
    ├── child task 3
    └── child task 4

`async let` 和 `TaskGroup` 都会产生 Child Task，只不过`async let`看起来不明显。


在任务树的情况下，在`withTaskCancellationHandler`的`onCancel`回调中，得到的当前Task并不是当前异步函数的Task，假设上面例子的`asyncRequest()`内注册了取消，对于上面`child task 1`的异步函数中回调`onCancel`闭包内的当前Task为顶级Task，对于`child task 3`的`onCancel`闭包内则是`child task 2`也就是TaskGroup。使用`Task.isCancelled`也是获取当前Task来判断，所以也符合这个情况。

如下两个代码位置打印当前Task的hashValue，在作为child task的情况下，打印是不一样的，位置1才是真正的当前Task。
```swift
func cancelableAsyncRequest() async throws -> String {
    // 代码位置1
    withUnsafeCurrentTask { task in
        print("CurrentTask \(task?.hashValue)")
    }

    return try await withTaskCancellationHandler {
    // ...
    } onCancel: {
        // 代码位置2
        withUnsafeCurrentTask { task in
            print("CurrentTask \(task?.hashValue)")
        }
    }
}
```

> **结构化并发的提案：https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md#async-let-to-create-child-tasks-within-a-scope**

## Sendable
`Sendable`是一个标记型的协议，不需要实现任何属性和方法，它用于描述可以安全共享的类型，在跨并发域传输时不会引发数据竞争问题。比如使用Task、async/await并发编程时，编译器会检测符合`Sendable`的类型，才能在并发域之间安全地传输。

例如这个代码会报错：
```swift
class MyClass {
    init(count: Int = 0) {
        self.count = count
    }
    var count: Int
}

func exampleFunc() async {
    let isNotSendable = MyClass()

    Task { // ❌ 报错：Sending value of non-Sendable type '() async -> ()' risks causing data races
        isNotSendable.count += 1 // Task内部访问
    }
    isNotSendable.count += 1 // Task外部访问
}
```
编译器会提示传输一个不符合`Sendable`的类型的值，会导致数据竞争问题。通俗地说就是Task内部和外部并发访问了同一个变量，显然是不安全的。

这里就可以推断出符合`Sendable`的类型支持并发安全，那么为什么符合`Sendable`的类型就能保证安全呢？我们先看符合`Sendable`的要求：
1. 标准库中的值类型都遵循了`Sendable`协议，因为值类型传递的是副本，本身是线程安全的。（比如Int、String、Bool等）
2. 如果包含的全部属性/元素都符合`Sendable`，`结构体`、`枚举`、`元组`、`Array/Set/Dictionary等集合`也自动隐式符合`Sendable`。
3. 元类型（例如 Int.Type，表达式 Int.self 生成的类型）始终符合 Sendable，因为它们是不可变的。
4. 不可变的引用类型（class），例如 final class 且只有 let 属性，可以主动标记为`Sendable`。
5. 可变的自定义引用类型，如果自己通过锁或其他机制来确保线程安全，也可以标记为`Sendable`，但此时编译器仍然会报错，所以需要加上`@unchecked`。

泛型类型要隐式遵循`Sendable`，需要它内部的元素符合`Sendable`，比如：
```swift
struct X<T: Sendable> {  // 隐式自动符合Sendable
  var value: T
}

struct Y<T> {    // 不能隐式自动符合Sendable，因为 T 不遵循 Sendable
  var value: T
}
```

其实可以看出来，符合`Sendable`的类型一般都是不可变的，因为不可变的类型自然就是线程安全的，如果是可变的情况就会要求开发者自行保证线程安全，这就是`Sendable`保证线程安全的设计思路。(结构体是值类型，所以它的属性是可变的也没关系，因为修改结构体的属性值就是创建新的副本)

> Int类型符合`Sendable`，但在源码中看不到，因为Int相关的代码是动态生成的：
https://github.com/swiftlang/swift/blob/main/stdlib/public/core/IntegerTypes.swift.gyb

### 一些Sendable的情况
```swift
✅ 符合Sendable，不显式标记也会自动符合Sendable
enum Repeness: Sendable {
    case hard
    case perfect
    case mushy(daysPast: Int)
}

✅ 符合Sendable，不显式标记也会自动符合Sendable
struct Pineapple: Sendable {
    var weight: Double
    var ripeness: Repeness
}

✅ 符合Sendable，不显式标记也会自动符合Sendable
struct Crate: Sendable {
    var pineapples: [Pineapple]
}

❌ 不符合Sendable的普通class
class Chicken {
    let name: String
    var currentHunger: HungerLevel

    func feed() { 

    }
}

❌ 不能标记为Sendable，因为它包含非Sendable状态，不能安全共享
struct Coop: Sendable {
    var flock: [Chicken]
}
```

要符合Sendable，就要求它内部的所有属性/元素都是符合`Sendable`的，这里`name`是String类型，符合`Sendable`，所以State枚举类型也符合`Sendable`。
```swift
enum State: Sendable {
    case loggedOut
    case loggedIn(name: String)
}
```

> `Sendable`可以由 Swift 自动推断，也可以显式声明`Sendable`（但要满足要求才能通过编译检查）

### 让Class符合Sendable
由于Class是引用类型，所以要让它符合`Sendable`，会比枚举、结构体等值类型要求更多：
1. 标记为`final`：避免创建子类添加了一个不可发送的属性，这时事情就会变得相当混乱。
2. 所有存储属性都是immutable和sendable
3. 没有父类，或者父类是`NSObject`

比如这个类可以符合`Sendable`：
```swift
final class MyClass: Sendable {
    init(count: Int = 0) {
        self.count = count
    }
    let count: Int
}
```

但如果类中确实需要有可变属性，可以使用`@unchecked`结合`Sendable`，此时编译器不会检查，所以需要开发者自行通过锁、队列等机制保证线程安全。当类型不能符合`Sendable`，但你确信你的实现是线程安全的，`@unchecked`属性非常有用，比如：
```swift
final class MyClass: @unchecked Sendable {
    private var value: Int = 0
    // 使用串行队列保证线程安全
    private let queue = DispatchQueue(label: "com.myapp.syncQueue")

    func updateValue(_ newValue: Int) {
        queue.sync {
            self.value = newValue
        }
    }
    
    func getValue() -> Int {
        return queue.sync { value }
    }
}
```

### Sendable 函数和闭包
`Sendable`协议在语法上只能用于类型声明，对于函数和闭包，则可以使用`@Sendable`属性（Swift 5 添加）或者`sending`（Swift 6 添加）。

我们看一下`Task.init`函数的声明：
```swift
// Swift 5 使用 @Sendable
public init(
  priority: TaskPriority? = nil,
  operation: @Sendable @escaping () async -> Success
)

// Swift 6 改为使用 sending
public init(
    priority: TaskPriority? = nil, 
    operation: sending @escaping @isolated(any) () async -> Success
)
```

### @Sendable的使用
首先了解一下`@Sendable`，使用`@Sendable`属性修饰的闭包，意味着该闭包中捕获的任何值的类型都需要是`Sendable`，这样才可以安全地跨并发边界传递和调用该闭包。

由于Task的init函数源码已经改为使用`sending`，即使把 Xcode 16 的 Swift 6 编译器设置为 Swift 5 模式，也仍然会使用`sending`的版本，所以要测试`@Sendable`的效果，我们可以自定义一个使用`@Sendable`的函数来测试。
```swift
public func sendableClosure(
    operation: @Sendable () -> Void
) {
    // ...
}

// 一个不符合Sendable的类
class MyClass {
    var value: Int = 0
}

func test() {
    let isNotSendable = MyClass()
    sendableClosure {
        // ❌ Capture of 'isNotSendable' with non-sendable type 'MyClass' in a `@Sendable` closure
        isNotSendable.value += 1
    }
}
```
此时报错，因为`sendableClosure`的闭包参数使用了`@Sendable`，捕获的值必须是`Sendable`。所以只要把`MyClass`改为符合`Sendable`的类型，就可以解决。

### sending的使用
最新的`Task.init`的闭包已经改为使用`sending`，所以我们验证`sending`的效果可以直接使用`Task.init`。
```swift
func test() {
    let isNotSendable = MyClass()
    Task {
        isNotSendable.value += 1
    }
}
```
虽然仍然是使用不符合`@Sendable`的类型，但此时不会报错了，因为在 Swift 6 中添加的`sending`允许捕获不符合`Sendable`的值，只要在捕获后不再在其他地方访问它。**毕竟不再在其他地方使用，也就不会有并发安全问题了**

所以假如其他地方使用就会报错了：
```swift
func test() {
    let isNotSendable = MyClass()
    Task {
        isNotSendable.value += 1
    }
    isNotSendable.value += 1 // ❌ 报错
}
```
因为这样就又产生了并发问题。

**所以使用`sending`的情况下，如果捕获`Sendable`类型的值，则没有限制，如果捕获非`Sendable`类型的值，则不能在其他地方再次访问**


### `@Sendable`和`sending`的注意点
`@Sendable`标记的函数/闭包可以安全地跨并发域传输（它隐式地符合`Sendable`协议），编译器会检查标记`@Sendable`的函数/闭包的几个方面：
1. `@Sendable`标记的函数/闭包捕获的任何值也必须符合`Sendable`。
2. `@Sendable`标记的函数/闭包只能使用按值捕获，`let`引入的不可变值的捕获是隐式的按值捕获，任何其他捕获都必须通过捕获列表指定

第2点可以通过如下代码验证：
```swift
func test() {
    // 这里使用 var 而不是 let
    var isSendable = MyClass()
    sendableClosure {
        isSendable.updateValue(12) // ❌ Reference to captured var 'isNotSendable' in concurrently-executing code
    }
}

func test() {
    var isSendable = MyClass()
    sendableClosure { [isSendable] in // ✅ 使用了捕获列表，显式按值捕获即可
        isSendable.updateValue(12)
    }
}
```

`sending`标记的闭包参数同样也要求按值捕获：
```swift
func test1() {
    var isNotSendable: MyClass? = MyClass()
    Task { [isNotSendable] in // ⚠️ 如果不显式使用捕获列表按值捕获，则无法使用 var 可变变量
        isNotSendable?.value += 1
    }
    isNotSendable = nil
}
```


`@Sendable`和`sending`除了前面说的功能略有差别，使用场景也略有差别：
* `@Sendable`：用于闭包/函数类型。显式要求所有捕获值都必须符合`Sendable`
* `sending`：用于参数和返回值。自动推导捕获是否安全（非Sendable值没有在其他地方访问也可以），并且要求**非Sendable的值**的所有权转移，其他地方不能再访问

**只能使用`@Sendable`的情况：**
```swift
// 只能用 @Sendable
typealias MyClosure = @Sendable () -> Void

// 只能用 @Sendable
struct MyStruct: Sendable {
    let value: Int
    @Sendable func performAction() {
        print("Action with \(value)")
    }
}
```

**只能使用`sending的情况**
```swift
// 只能使用 sending
func sendingFunc(data: sending MyClass) {

}
```

**`@Sendable`和`sending`都可以使用的情况：**
```swift
// 可以用 sending
func runInBackground(_ work: sending @escaping () -> Void) {
    Task { work() }
}

// 也可以用 @Sendable
func runInBackground(_ work: @Sendable @escaping () -> Void) {
    Task { work() }
}
```

Sendable 并不意味着原子性，它只保证值可以安全地跨任务和线程发送

## Actor
Actor是一种并发模型，它的核心思想是独立维护隔离（isolated）状态，外部无法直接访问，actor会持有消息队列，外部读写actor中的可变状态都会发送消息到actor的消息队列，然后被Actor中的Executor串行执行，从而保证actor内的状态是线程安全的。比如Kotlin协程中的Actor实现，本质上就是一个消费者Channel（SendChannel）。

actor 确保其内部状态只能被一个任务（线程）一次访问，从而消除数据竞争。传统的并发工具（如锁、信号量）使用复杂且容易出错，actor 提供了一种更高层次的抽象，简化了并发编程。

在前面介绍`Sendable`时，如果有可变属性的Class要满足`Sendable`协议，就需要使用`@unchecked`忽略检查，并且自行保证线程安全。但如果每个都要这样改写，还是有点麻烦的，此时就可以使用Swift提供的`actor`来声明一个引用类型：
```swift
actor Counter {
    var value = 0
    func increment() { value += 1 }
}
```

编译器会自动对其处理，让它符合actor模型（估计本质还是类，只不过添加了消息队列并发保护等机制）。用`actor`声明的引用类型都会隐式符合`Actor`协议，自然也就符合`Sendable`协议：
```swift
protocol Actor : AnyObject, Sendable
```

`actor`类型可以安全地跨并发域使用，但访问时需要使用`await`关键字，`await`会让访问变为通过消息队列异步执行。
```swift
func test() {
    let counter = Counter()

    Task {
        await counter.increment()
    }
}
```

### isolated
假设我们有一个使用`actor`编写的银行账户类型：
```swift
actor BankAccount {
    enum BankError: Error {
        case insufficientFunds
    }
    
    var balance: Double
    
    init(initialDeposit: Double) {
        self.balance = initialDeposit
    }
    
    // 取钱
    func withdraw(amount: Double) throws {
        guard balance >= amount else {
            throw BankError.insufficientFunds
        }
        balance -= amount
    }
    
    // 存钱
    func deposit(amount: Double) {
        balance = balance + amount
    }
}

extension BankAccount {

  // 转账方法
  func transfer(amount: Double, to other: BankAccount) throws {
    try withdraw(amount: amount)
    other.balance = other.balance + amount  // ❌ 报错：actor隔离属性 'balance' 只能在 actor 内部访问
  }
}
```
这里的`transfer`方法直接访问了另一个`BankAccount`的`balance`属性，这是不允许的，因为`actor`的隔离状态只允许在actor内部访问，不能在其他地方访问。

此时我们可以不直接访问`balance`，而是调用它的`deposit`方法，在其中修改`balance`属性，不过此时需要添加`await`才能访问，所以`transfer`方法也需要显式加上`async`：
```swift
func transfer(amount: Double, to other: BankAccount) async throws {
    try withdraw(amount: amount)
    await other.deposit(amount: amount)
}
```
此时修改其他对象的`balance`属性改为通过`await`调用其他对象的`deposit`方法来完成，就可以避免在外部直接访问actor对象的属性的问题。


我们还可以使用`isolated`标记方法中的Actor类型的参数，使当前方法变为该Actor类型参数的隔离域，在方法内就可以直接访问该参数的属性了，并且不需要`await`：
```swift
func transfer(amount: Double, to other: isolated BankAccount) async throws {
    try await withdraw(amount: amount)
    other.balance = other.balance + amount
}
```
不过由于此时方法内变为了`other`对象的隔离域，所以访问当前对象的`withdraw`方法就需要加上`await`了，这就要看业务情况了，如果方法内只需要操作非当前对象的单个actor对象，那么通过`isolated`就可以在方法内部直接访问属性，并且不需要`await`。

----------
#### isolated的使用限制
一个声明（比如函数、变量等）不能同时被隔离到全局 actor（如 @MainActor）和实例 actor（如某个具体的 actor 类型实例）。这是为了避免隔离规则的冲突和复杂性。
```swift
actor Counter {
  var value = 0
  
  @MainActor func updateUI(view: CounterView) async {
    view.intValue = value  // ❌ 错误：`value` 是隔离到 `Counter` 实例的，但当前上下文是 `@MainActor` 隔离的
    view.intValue = await value // ✅ 正确：通过异步方式读取 `value`
  }
}
// Counter 是一个 actor 类型，value 是它的属性，默认隔离到 Counter 实例。updateUI 方法被标注为 @MainActor，意味着它的执行上下文被隔离到全局的 MainActor（通常用于主线程的 UI 更新）。正确的方式是使用 await 异步访问 value，因为 await 允许跨越隔离域安全地访问 actor 的数据。
```

任何函数类型都不能同时包含`isolated`参数和`global actor`限定符，毕竟这样就冲突了，无法知道到底应该隔离到`MainActor`，还是隔离到`Counter`的隔离域 ：
```swift
@MainActor func tooManyActors(counter: isolated Counter) { } // ❌ error: 'isolated' parameter on a global-actor-qualified function
```

#### 在 deinit 函数使用 isolated
`deinit`函数支持标记`isolated`，[提案文档](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0371-isolated-synchronous-deinit.md)
```swift
@MainActor
class Foo {
  isolated deinit {} // Isolated on MainActor.shared
}

actor Bar {
  isolated deinit {} // Isolated on self
}
```

> **Xcode的 Edit Scheme -> Run -> Diagnostics -> Thread Sanitizer 打开这个，可以在运行时检查数据竞争问题**

#### 通过`isolated`减少await
使用`isolated`可以在方法内部不使用`await`，只需要外部调用方法时使用一次`await`即可，

```swift
actor Counter {
    private var count = 0

    func increment() {
        count += 1
    }

    func getCount() -> Int {
        count
    }
}

// 接收 isolated 参数的闭包
func runInActor(_ callback: (isolated Counter) async -> Void) async {
    let counter = Counter()
    await callback(counter)
}

func test() {
    Task {
        await runInActor { isolatedCounter in
            isolatedCounter.increment()
            print(isolatedCounter.getCount())
        }
    }
}
```
仅调用一次`await`，闭包内部则可以直接访问`Actor`。


### nonisolated
如果某些方法不需要隔离保护，比如并不涉及 actor 的状态，可以标记为 `nonisolated`，这样它们就不受隔离约束，可以直接同步调用，不需要 `await`。这样也可以提高性能，避免不必要的隔离域切换。
```swift
actor BankAccount {

    let accountNumber: String

    nonisolated var details: String {
        "Account Number: \(accountNumber)"
    }

    // ...
}

func test() {
    Task {
        print(account.details) // 这里没有使用 await
    }
}
```

## 线程调度
前面介绍`Task.init`方法时，没有涉及另一个重载函数，它提供了`executorPreference`参数，允许我们指定一个`TaskExecutor`线程调度器：
```swift
init(
    executorPreference taskExecutor: consuming (any TaskExecutor)?,
    priority: TaskPriority? = nil,
    operation: sending @escaping () async -> Success
)
```
指定`TaskExecutor`后，`Task`的`operation`代码块会被调度到指定的线程中执行。

`TaskExecutor`的父协议为`Executor`，我们先了解一下它：
```swift
// 源码：https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/Executor.swift

protocol Executor : AnyObject, Sendable {

    @available(SwiftStdlib 5.1, *)
    func enqueue(_ job: UnownedJob)

    @available(StdlibDeploymentTarget 5.9, *)
    func enqueue(_ job: consuming ExecutorJob)

    @available(StdlibDeploymentTarget 5.9, *)
    @available(*, deprecated, message: "Implement 'enqueue(_: consuming ExecutorJob)' instead")
    func enqueue(_ job: consuming Job)
}

// 默认实现
extension Executor {

  public func enqueue(_ job: UnownedJob) {
    self.enqueue(ExecutorJob(job))
  }

  @available(StdlibDeploymentTarget 5.9, *)
  public func enqueue(_ job: consuming ExecutorJob) {
    self.enqueue(Job(job))
  }

  public func enqueue(_ job: consuming Job) {
    self.enqueue(UnownedJob(job))
  }
}
```
这三个实现是相互递归的，所以开发者自定义`Executor`时，必须至少重写其中一个方法，打破死循环递归。三个重载方法的Job类型分别如下：
* `Job`：已经标记了Deprecated，可以忽略它
* `UnownedJob`：从源码看本质上是对 C++ Runtime 层的`Builtin.Job`指针的包装，实际的任务调度执行都是在 C++ Runtime 层完成，Swift层ARC并不会对`Builtin.Job`执行`retain`、`release`之类的引用计数管理，`UnownedJob`只是临时引用`Builtin.Job`指针，不拥有它的生命周期（所以这可能就是命名为`UnownedJob`的原因）。
* `ExecutorJob`：从源码看和`UnownedJob`没什么明显区别（Swift层的`Job`源码也类似），它的出现是因为Swift在 5.9 版本新增了move-only类型，也就是不可复制类型，所以可以使用`comsuming`关键字来标记，确保`ExecutorJob`在被使用后就会被销毁，不会被重复使用。这样的语法和语义更适合这个场景，所以开发者应该优先使用`ExecutorJob`。[0392-custom-actor-executors官方提案](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0392-custom-actor-executors.md)中提到将`Job`重命名为`ExecutorJob`，使其不太可能与现有类型名称发生冲突，并且为了向后兼容，UnownedJob类型仍然存在，例如`ExecutorJob`在有些场景的限制。
。

**所以下文的介绍中我们会优先使用`ExecutorJob`，但些情况需要借助`UnownedJob`**

### 自定义`TaskExecutor`
`Task.init`方法可以传入`TaskExecutor`类型的参数，用于指定代码块执行的线程。首先看一下`TaskExecutor`的源码：
```swift
// https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/Executor.swift

// 忽略了废弃的 Job 类型重载方法
@available(StdlibDeploymentTarget 6.0, *)
public protocol TaskExecutor: Executor {

    func enqueue(_ job: UnownedJob)

    func enqueue(_ job: consuming ExecutorJob)

    // 转换为 UnownedTaskExecutor，extension 中有默认实现
    func asUnownedTaskExecutor() -> UnownedTaskExecutor
}

@available(StdlibDeploymentTarget 6.0, *)
extension TaskExecutor {
  public func asUnownedTaskExecutor() -> UnownedTaskExecutor {
    // 默认实现就是直接使用 UnownedTaskExecutor 包装一下
    unsafe UnownedTaskExecutor(ordinary: self)
  }
}
```
实现`TaskExecutor`需要实现一个`enqueue`方法，我们一般选择实现参数类型为`ExecutorJob`的`enqueue`方法。

那么再看下`ExecutorJob`的代码：
```swift
public struct ExecutorJob: Sendable, ~Copyable {
  internal var context: Builtin.Job

  @usableFromInline
  internal init(context: __owned Builtin.Job) {
    self.context = context
  }

  public init(_ job: UnownedJob) {
    self.context = job._context
  }

  public init(_ job: __owned Job) {
    self.context = job.context
  }

  // ......
}

extension ExecutorJob {
  __consuming public func runSynchronously(on executor: UnownedSerialExecutor) {
    unsafe _swiftJobRun(UnownedJob(self), executor)
  }

  // 在调用线程上直接运行 job，阻塞至 job 完成。此方法的预期用途是让 Executor 决定何时何地调用从而运行 job。
  // 传入的 Executor 用于建立 job 的 Executor 上下文，并且应与语义上调用 runSynchronously 方法的 Executor 相同。
  // 此操作会消耗job，防止其在运行后被意外使用，将 ExecutorJob 转换为 UnownedJob 并多次对其调用runSynchronously是
  // 未定义的行为，因为job只能运行一次，并且在运行后不得访问。
  @available(StdlibDeploymentTarget 6.0, *)
  __consuming public func runSynchronously(on executor: UnownedTaskExecutor) {
    unsafe _swiftJobRunOnTaskExecutor(UnownedJob(self), executor)
  }

  @available(StdlibDeploymentTarget 6.0, *)
  __consuming public func runSynchronously(isolatedTo serialExecutor: UnownedSerialExecutor,
                               taskExecutor: UnownedTaskExecutor) {
    unsafe _swiftJobRunOnTaskExecutor(UnownedJob(self), serialExecutor, taskExecutor)
  }
}
```
`ExecutorJob`的核心方法就是`runSynchronously`（暂时可以只关心仅有`UnownedTaskExecutor`参数的方法），调用它就会同步执行job，阻塞当前线程，直到job执行完成。

> `runSynchronously`方法都标记了`__consuming`属性，它是Swift正式提供给开发者使用`consuming`关键字之前的内部实现，估计是考虑兼容性、ABI稳定性，所以保留下来了。（可以在[源码](https://github.com/swiftlang/swift/blob/main/include/swift/AST/DeclAttr.def)中找到`__consuming`和`consuming`都存在）

综合`TaskExecutor`和`ExecutorJob`的代码，我们现在来自定义一个`TaskExecutor`：
```swift
final class MyTaskExecutor: TaskExecutor {
    let queue: DispatchQueue

    init(queue: DispatchQueue) {
        self.queue = queue
    }

    func enqueue(_ job: consuming ExecutorJob) {
        print("queue ExecutorJob")
        // 因为不可复制类型 ExecutorJob 不能放到逃逸闭包中，所以需要转换为 UnownedJob 类型
        let job = UnownedJob(job)
        queue.async {
            job.runSynchronously(on: self.asUnownedTaskExecutor())
        }
    }
}
```
使用时可以给`MyTaskExecutor`传入一个`DispatchQueue`，`MyTaskExecutor`会将`ExecutorJob`添加到`DispatchQueue`中执行。
```swift
func asyncFunc(key: String) async {
    print("asyncFunc\(key) run on thread: \(Thread.currentThread)")
}

// 传入 DispatchQueue.global()，则会在在全局线程中执行
let executor = MyTaskExecutor(queue: DispatchQueue.global())
Task(executorPreference: executor) {
    print("task run on thred: \(Thread.currentThread)")
    // 并发执行3个异步函数
    await withDiscardingTaskGroup{ group in
        group.addTask {
            await asyncFunc(key: "1")
        }
        group.addTask {
            await asyncFunc(key: "2")
        }
        group.addTask {
            await asyncFunc(key: "3")
        }
    }
}
// 打印：
// task run on thred:  <NSThread: 0x600003a4c780>{number = 2, name = (null)}
// asyncFunc1 run on thread: <NSThread: 0x600003a4c780>{number = 2, name = (null)}
// asyncFunc2 run on thread: <NSThread: 0x600003a4c780>{number = 2, name = (null)}
// asyncFunc3 run on thread: <NSThread: 0x600003a481c0>{number = 3, name = (null)}


// 传入 DispatchQueue.main，会在主线程中执行
let executor = MyTaskExecutor(queue: DispatchQueue.main)
Task(executorPreference: executor) {
    print("task run on thred: \(Thread.currentThread)")
    await asyncFunc(key: "1")
}
// 打印：
// task run on thred:  <_NSMainThread: 0x6000003d41c0>{number = 1, name = main}
// asyncFunc1 run on thread: <_NSMainThread: 0x6000003d41c0>{number = 1, name = main}
```

### 使用`TaskExecutor`的场景
[0417-task-executor-preference提案](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0417-task-executor-preference.md)

在上面👆实现`TaskExecutor`的例子中，使用了包含`executorPreference`参数的`Task.init`方法，`Task`内的代码全都会通过`TaskExecutor`调度到特定线程中执行。而除了直接在`Task.init`中指定`executorPreference`外，还可以使用`TaskGroup.addTask`和`withTaskExecutorPreference`来指定`TaskExecutor`。

#### `TaskGroup.addTask`指定`TaskExecutor`
```swift
Task(executorPreference: MyTaskExecutor(queue: DispatchQueue.global())) {
    print("task run on thred: \(Thread.currentThread)")
    await withDiscardingTaskGroup { group in
        group.addTask(executorPreference: MyTaskExecutor(queue: DispatchQueue.global())) {
            await asyncFunc(key: "1")
        }
        group.addTask(executorPreference: MyTaskExecutor(queue: DispatchQueue.main)) {
            await asyncFunc(key: "2")
        }
        group.addTask(executorPreference: nil) {
            await asyncFunc(key: "3")
        }
    }
}
// 打印：
// task run on thred:  <NSThread: 0x600000ce4000>{number = 2, name = (null)}
// asyncFunc1 run on thread: <NSThread: 0x600000ce00c0>{number = 3, name = (null)}
// asyncFunc2 run on thread: <_NSMainThread: 0x600000ce84c0>{number = 1, name = main}
// asyncFunc3 run on thread: <NSThread: 0x60000020c080>{number = 4, name = (null)}
```
`TaskGroup.addTask`方法设置`taskExecutor`后，任务会通过指定的`TaskExecutor`执行。如果传`nil`就相当于调用没有`taskExecutor`参数的重载方法，继承当前外部上下文的`executor preference`。

#### `withTaskExecutorPreference`指定`TaskExecutor`
```swift
Task(executorPreference: MyTaskExecutor(queue: DispatchQueue.global())) {
    print("task run on thread: \(Thread.currentThread)")
    await withTaskExecutorPreference(MyTaskExecutor(queue: DispatchQueue.main)) {
        await asyncFunc(key: "1")
    }
    await withTaskExecutorPreference(nil) {
        await asyncFunc(key: "2")
    }
}
// 打印：
// task run on thread: <NSThread: 0x6000011bc080>{number = 2, name = (null)}
// asyncFunc1 run on thread: <_NSMainThread: 0x6000011b41c0>{number = 1, name = main}
// asyncFunc2 run on thread: <NSThread: 0x600002a2c4c0>{number = 3, name = (null)}
```
`withTaskExecutorPreference`方法会将`operation`代码块和它内部创建的`child task`调度在传入的`Executor`中执行。如果传入`nil`则不会改变当前的`executor preference`。

`executorPreference`的继承规则：
1. 继承的情况：
    * TaskGroup 的 addTask（）， 除非被显式参数覆盖
    * async let
    * Default Actors（未自定义SerialExecutor）上的方法
2. 不继承的情况：
    * 非结构化任务：Task {} 和 Task.detached {}
    * actors上使用自定义执行器的方法（包括 MainActor 等）

### 自定义`actor`和`SerialExecutor`
前面介绍的`actor`类型，本质上是一个语法糖，它隐式实现了`Actor`协议：
```swift
// https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/Actor.swift
public protocol Actor : AnyObject, Sendable {

    nonisolated var unownedExecutor: UnownedSerialExecutor { get }
}
```
这里有一个`unownedExecutor`属性，actor类型的方法将通过它调度执行，在我们没有自定义的情况下，会有一个默认实现，它会调度给全局并发队列执行。

我们也可以自定义一个`SerialExecutor`（大多数情况没必要自定义），首先了解`SerialExecutor`协议的定义：
```swift
public protocol SerialExecutor : Executor {

    func enqueue(_ job: consuming ExecutorJob)

    // 将此executor值转换为借用executor引用的优化形式
    func asUnownedSerialExecutor() -> UnownedSerialExecutor

    func isSameExclusiveExecutionContext(other: Self) -> Bool

    func checkIsolated()
}

extension SerialExecutor {
  public func asUnownedSerialExecutor() -> UnownedSerialExecutor {
    unsafe UnownedSerialExecutor(ordinary: self)
  }
  
  public func isSameExclusiveExecutionContext(other: Self) -> Bool {
    return self === other
  }
}
```

我们一般只需要重写`enqueue`方法，并且在`actor`类型中使用自定义的`SerialExecutor`，例如：
```swift
final class MySerialExecutor: SerialExecutor {

    let queue = DispatchQueue.main // 这里故意让它在主线程执行

    func enqueue(_ job: consuming ExecutorJob) {
        let unownedJob = UnownedJob(job)
        queue.async {
            unownedJob.runSynchronously(on: self.asUnownedSerialExecutor())
        }
    }
}

actor MyActor {
    static let executor = MySerialExecutor()

    var count = 0

    func performTask() async {
        print("performTask  start --------- \(count)")
        print("performTask  current thread: \(Thread.currentThread)")
        var i = 0
        while i < 10000 {
            i += 1
            count += 1
        }
        print("performTask  end ------------ \(count)")
    }

    nonisolated var unownedExecutor: UnownedSerialExecutor {
        MyActor.self.executor.asUnownedSerialExecutor()
    }
}

let actor = MyActor()
Task {
    await actor.performTask()
}
Task {
    await actor.performTask()
}
Task {
    await actor.performTask()
}

// 打印结果：
// performTask  start --------- 0
// performTask  current thread: <_NSMainThread: 0x6000035381c0>{number = 1, name = main}
// performTask  end ------------ 10000
// performTask  start --------- 10000
// performTask  current thread: <_NSMainThread: 0x6000035381c0>{number = 1, name = main}
// performTask  end ------------ 20000
// performTask  start --------- 20000
// performTask  current thread: <_NSMainThread: 0x6000035381c0>{number = 1, name = main}
// performTask  end ------------ 30000
```
`actor`的方法调用全部执行在了自定义的`MySerialExecutor`使用的`DispatchQueue.main`中。

**但假如自定义的`SerialExecutor`调度执行任务使用`DispatchQueue.global()`之类的并发队列方式，破坏了`SerialExecutor`的串行性质，那么`actor`类型的方法执行也就不能保证串行执行和线程安全了。**

> 另外，上面可以看到实际会使用`UnownedSerialExecutor`类型，它是一个特殊的结构体，它内部不是直接持有`SerialExecutor`的强引用，而是用一个底层的指针（`Builtin.Executor`）不增加引用计数地引用原始的`SerialExecutor`对象。这样，调度系统在传递、存储 `UnownedSerialExecutor`时，不再对原始的对象进行 retain/release 操作，从而大大减少了 ARC 带来的性能开销。

### 同时实现`TaskExecutor`和`SerialExecutor`
我们也可以同时实现`TaskExecutor`和 `SerialExecutor`，这样的`Executor`能够既被`actor`使用，又可以用于设置`Task`。
```swift
final class NaiveQueueExecutor: TaskExecutor, SerialExecutor {
    let queue: DispatchQueue

    init(_ queue: DispatchQueue) {
        self.queue = queue
    }

    public func enqueue(_ _job: consuming ExecutorJob) {
        let job = UnownedJob(_job)
        queue.async {
            job.runSynchronously(
                isolatedTo: self.asUnownedSerialExecutor(),
                taskExecutor: self.asUnownedTaskExecutor()
            )
        }
    }

    // asUnownedSerialExecutor 和 asUnownedTaskExecutor 可以使用默认实现，不用重写
}
```

### 自定义`GlobalActor`
前面介绍的`actor`类型可以把状态操作包装在这个`actor`实例中，对它的访问会调度到该`actor`的`serial executor`中串行执行。但如果需要保护调度的代码逻辑分散各处，使用自定义`actor`类型并不方便，此时就可以使用`GlobalActor`。

`GlobalActor`协议定义如下，它的核心是包含一个名为`shared`的`Actor`类型的静态属性：
```swift
public protocol GlobalActor {

  associatedtype ActorType: Actor

  // 全局共享的actor实例，用于为使用自定义GlobalActor注解的提供互斥访问，
  // 也就是说，其他代码访问使用此 GlobalActor 标记的声明，会通过这个actor
  static var shared: ActorType { get }

  // 上面shared实例的unownedExecutor，有默认实现，一般不用重写
  static var sharedUnownedExecutor: UnownedSerialExecutor { get }
}

extension GlobalActor {
  // sharedUnownedExecutor 的默认实现
  public static var sharedUnownedExecutor: UnownedSerialExecutor {
    unsafe shared.unownedExecutor
  }
}
```
可以看出来，自定义一个`GlobalActor`重点就是提供一个`shared`实例，也就是一个`Actor`类型的对象。另外，自定义的`GlobalActor`还需要配合`@globalActor`修饰，才能使用。

那么我们来自定义一个`GlobalActor`，它可以是`struct`、`enum`、`actor`或者`final class`：
```swift
// 自定义 SerialExecutor 类
public final class CustomExecutor: SerialExecutor {
    public func enqueue(_ job: consuming ExecutorJob) {
        let job = UnownedJob(job)
        DispatchQueue.main.async {
            job.runSynchronously(on: self.asUnownedSerialExecutor())
        }
    }
}

// 自定义 actor 类
public actor CustomActor {
    static let executor = CustomExecutor()

    // 使用自定义的 executor，否则会在默认的 DefaultActor 全局并发队列串行执行
    public nonisolated var unownedExecutor: UnownedSerialExecutor {
        CustomActor.executor.asUnownedSerialExecutor()
    }
}

// 自定义 GlobalActor 类
@globalActor public struct MyMainActor: GlobalActor {
    public static let shared = CustomActor()
}
```
定义了`CustomActor`和它的`CustomExecutor`，调度任务到主线程执行，然后在`MyMainActor`中使用`CustomActor`作为`shared`，同时给`MyMainActor`添加了`@globalActor`修饰，这样它就可以给类、函数、属性和闭包标记了。

使用方式如下：
```swift
@MyMainActor // 标记 @MyMainActor
func testAsync() async {
    print("testAsync  \(Thread.currentThread)")
}

Task {
    await testAsync()
}

let executor = MyTaskExecutor(queue: DispatchQueue(label: "xxx"))
Task(executorPreference: executor) {
    await testAsync()
}
```
`testAsync()`标记了我们自定义的`@MyMainActor`，所以无论`Task`在哪个线程/executor执行，`testAsync()`都会在主线程执行。

#### 简化自定义方式
上面自定义的`GlobalActor`和它使用的`Actor`类型是分开定义的，但各种类型都可以实现`GlobalActor`协议，所以可以直接用`acotr`类型来实现`GlobalActor`协议，这样就可以少自定义一个`Actor`类型。
```swift
public final class CustomExecutor: SerialExecutor {
    public func enqueue(_ job: consuming ExecutorJob) {
        let job = UnownedJob(job)
        DispatchQueue.main.async {
            job.runSynchronously(on: self.asUnownedSerialExecutor())
        }
    }
}

@globalActor public actor MyMainActor: GlobalActor {
    public static let shared = MyMainActor()

    public static let executor = CustomExecutor()

    public nonisolated var unownedExecutor: UnownedSerialExecutor {
        MyMainActor.executor.asUnownedSerialExecutor()
    }
}
```

#### `MainActor`
[MainActor源码](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/MainActor.swift)

实际上，官方已经提供了一个`GlobalActor`的实现类`MainActor`，它用来调度任务到主线程执行。它的源码如下：

```swift
@globalActor public final actor MainActor: GlobalActor {
  public static let shared = MainActor()

  @inlinable
  public nonisolated var unownedExecutor: UnownedSerialExecutor {
    return unsafe UnownedSerialExecutor(Builtin.buildMainActorExecutorRef())
  }

  @inlinable
  public static var sharedUnownedExecutor: UnownedSerialExecutor {
    return unsafe UnownedSerialExecutor(Builtin.buildMainActorExecutorRef())
  }

  @inlinable
  public nonisolated func enqueue(_ job: UnownedJob) {
    _enqueueOnMain(job)
  }
}

extension MainActor {

  /// Execute the given body closure on the main actor.
  @_alwaysEmitIntoClient
  public static func run<T: Sendable>(
    resultType: T.Type = T.self,
    body: @MainActor @Sendable () throws -> T
  ) async rethrows -> T {
    return try await body()
  }
}
```
`MainActor`的实现方式和我们刚才自定义的`MyMainActor`类似，只不过它使用的`Executor`是从`Runtime`层获取的。另外，它还提供了`MainActor.run`的方法用于将一个闭包运行在`MainActor`中。

#### 使用`GlobalActor`修饰的属性、方法、类
使用自定义`GlobalActor`标识的函数或属性声明只能从同一个`GlobalActor`的声明中同步访问，否则需要使用`await`异步访问：
```swift
@MainActor var globalTextSize: Int

@MainActor func increaseTextSize() { 
  globalTextSize += 2   // okay: 可以同步访问，因为都在 @MainActor 中
}

func notOnTheMainActor() async {
  globalTextSize = 12  // error: 无法同步访问，因为 globalTextSize 在 @MainActor 中
  increaseTextSize()   // error: 无法同步访问，因为 increaseTextSize() 在 @MainActor 中
  await increaseTextSize() // okay: 可以从任意代码位置异步访问executes there
}
```

`GlobalActor`可以用于闭包：
```swift
callback = { @MainActor in
  // ...
}

Task.detached { @MainActor in
  // ...
}
```

类可以用`global actor`标记，它的所有方法、属性和下标都将隐式隔离到该`global actor`，该类的任何不想隔离到`global actor`的成员都可以使用`nonisolated`修饰符选择退出。 
```swift
@MainActor
class IconViewController: NSViewController {
  @objc private dynamic var icons: [[String: Any]] = [] // implicitly @MainActor
    
  var url: URL? // implicitly @MainActor
  
  private func updateIcons(_ iconArray: [[String: Any]]) { // implicitly @MainActor
    icons = iconArray
        
    // Notify interested view controllers that the content has been obtained.
    // ...
  }
  
  nonisolated private func gatherContents(url: URL) -> [[String: Any]] {
    // ...
  }
}
```

#### `Global Actor`函数类型
在函数类型上使用`global actor`，可以让它仅能在特定的`global actor`上被使用：
```swift
var callback: @MainActor (Int) -> Void
```

值可以从没有`global actor`限定符的函数类型转换为具有`global actor`限定符的函数，例如：
```swift
func acceptInt(_: Int) { } // not on any actor

callback = acceptInt // okay: conversion to @MainActor (Int) -> Void
```
但不能反过来转换：
```swift
let fn3: (Int) -> Void = callback // error: removed global actor `MainActor` from function type
```
**因为将没有限制的`acceptInt`赋值给有限制的`callback`是没有影响的，但反过来的话，让有限制的类型赋值给没有限制的类型，显然会导致错误**

但是，当转换的结果是异步函数时 ，允许删除`global actor`限定符。在这种情况下，异步函数在执行其主体之前将首先转到`global actor`：
```swift
let callbackAsynchly: (Int) async -> Void = callback   // okay: implicitly hops to main actor

// 这可以被认为是以下的语法糖：
let callbackAsynchly: (Int) async -> Void = {
  await callback() // `await` is required because callback is `@MainActor`
}
```

#### `global actor`自动推断
1. 子类继承`@MainActor`标记的父类，则子类隐式被`@MainActor`标记
2. 重写父类使用`@MainActor`标记的方法，也隐式

还有这种规则，感觉不用记，使用时试一下就知道了。
```swift
@MainActor protocol P {
  func updateUI() { } // implicitly @MainActor
}

class C: P { } // C is implicitly @MainActor

// source file D.swift
class D { }

// different source file D-extensions.swift
extension D: P { // D is not implicitly @MainActor
  func updateUI() { } // okay, implicitly @MainActor
}
```

[参考官方提案0316-global-actors](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md)

> 比如iOS，`@MainActor`标记在`UIViewController`上，所以在实现类内部声明的方法默认都会在`@MainActor`执行，macOS的Appkit也是如此。

### 线程调度优先级
Swift 并发框架在线程调度方面的设计，感觉比 Kotlin 协程要难以理解一些。因为除了需要考虑`executorPreference`，还要考虑当前上下文的`actor isolation`。

#### 顶级变量和函数
首先来看**顶级变量和函数**的情况，根据Swift官方提案：
1. Swift 的 main 函数会隐式地在`MainActor`上运行（[官方提案0323-async-main-semantics](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0323-async-main-semantics.md)），
2. 顶级全局变量也会被隐式分配`@MainActor`隔离，以防止数据竞争，但顶级函数不会默认分配`@MainActor`（[官方提案0343-top-level-concurrency](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0343-top-level-concurrency.md)）。

```swift
// 在顶级作用域
var top = 1 // 隐式分配 @MainActor

func topFunc() { // 没有分配 actor，也就是 nonisolated
    // top = 2 🚫 不能直接访问 top
}

func topAsyncFunc() async {
    let a = await top // 只能通过异步获取 top
}

// 如果要修改 top，则需要隔离到相同的 actor 中，也就是 @MainActor。例如：
@MainActor
func topAsyncFunc1() async {
    top = 2
}

// 或者
func topAsyncFunc2() async {
    await MainActor.run{
        top = 2
    }
}

// 或者
func topAsyncFunc3() {
    Task { @MainActor in
        top = 2
    }
}
```

另外，如果在顶级作用域直接创建`Task`：
```swift
// 隐式分配 @MainActor
Task {
    top = 2
}
```
在顶级作用域直接调用`Task.init`创建Task实例，也是顶级作用域的变量，所以也会被隐式分配`@MainActor`隔离。

#### `actor isolation`和`taskPreference`的优先级
```swift
// 使用 MainActor 隔离
@MainActor
func mainTest() {
    print("mainTest: \(Thread.currentThread)")
    Task {
        print("mainTest task 1: \(Thread.currentThread)")
    }
    let taskExecutor = MyTaskExecutor(queue: DispatchQueue.global())
    Task(executorPreference: taskExecutor) {
        print("mainTest task 2: \(Thread.currentThread)")
    }
}

// test 没有 actor 隔离
func test() {
    print("test: \(Thread.currentThread)")
    Task {
        print("test task 1: \(Thread.currentThread)")
    }
    let taskExecutor = MyTaskExecutor(queue: DispatchQueue.main)
    Task(executorPreference: taskExecutor) {
        print("test task 2: \(Thread.currentThread)")
    }
}

test()
mainTest()

// 打印结果：
// test: <_NSMainThread: 0x600000a200c0>{number = 1, name = main}
// test task 1: <NSThread: 0x600000a20300>{number = 2, name = (null)}
// queue ExecutorJob 1
// mainTest: <_NSMainThread: 0x600000a200c0>{number = 1, name = main}
// queue ExecutorJob 1
// mainTest task 2: <NSThread: 0x600000a20300>{number = 2, name = (null)}
// test task 2: <_NSMainThread: 0x600000a200c0>{number = 1, name = main}
// mainTest task 1: <_NSMainThread: 0x600000a200c0>{number = 1, name = main}
```
以上代码，得出如下结论：
1. 没有 actor 隔离上下文时：
  * Task 没有设置 executorPreference，则运行在**全局并发队列**中（可以通过源码验证）
  * Task 设置了 executorPreference，则运行在对应的 taskExecutor 队列中
2. 有 actor 隔离上下文时：
  * Task 没有设置 executorPreference，则运行在 actor 隔离上下文中
  * Task 设置了 executorPreference，则运行在对应的 taskExecutor 队列中


另外，再验证结合 actor 对象的情况，actor 类型分为`Default Actor`和`自定义了 sirialExecutor 的 Actor`。
```swift
// 没有自定义 sirialExecutor 的默认 Actor
actor MyDefaultActor {
    var count = 0
    func performTask(_ key: String = "") {
        print("performTask \(key) thread: \(Thread.currentThread)")
        count += 1
    }
}

// @MainActor 无论是否配置 actor 隔离
func testDefaultActor() {
    print("testDefaultActor thread: \(Thread.currentThread)")

    let actor = MyDefaultActor()

    Task {
        await actor.performTask("1")
    }

    let taskExecutor = MyTaskExecutor(queue: DispatchQueue.main)
    Task(executorPreference: taskExecutor) {
        await actor.performTask("2")
    }
}

testDefaultActor()

// 打印结果：
// testDefaultActor thread: <_NSMainThread: 0x6000003dc600>{number = 1, name = main}
// performTask 1 thread: <NSThread: 0x6000003d8140>{number = 2, name = (null)}
// performTask 2 thread: <_NSMainThread: 0x6000003dc600>{number = 1, name = main}
```
**Default Actor类型，会通过全局并发队列执行，但如果所在`Task`设置了`executorPreference`，则会受它影响。** 

----------

在验证的时候发现，同一个`actor`实例（没有自定义`SerialExecutor`），多个并发调用时间重叠（前一个还在执行，下一个就开始await），会导致并发调用的第一个方法执行的线程成为其他重叠的调用所在的线程。不确定是bug还是swift为了性能之类的目的故意为之。
```swift
actor MyDefaultActor {
    var count = 0
    func performTask(_ key: String = "") {
        print("performTask \(key) thread: \(Thread.currentThread)")
        // 增大耗时，更容易触发并发调用的重叠
        for _ in 1...500000 {
            count += 1
        }
    }
}

func testDefaultActor() {
    print("testDefaultActor thread: \(Thread.currentThread)")

    let actor = MyDefaultActor()

    Task {
        await actor.performTask("1") // <NSThread: 0x600003a100c0>{number = 2, name = (null)}
    }

    let taskExecutor = MyTaskExecutor(queue: DispatchQueue.main)
    // ⚠️ 虽然设置了主线程的 executor，但却在非主线程执行了
    Task(executorPreference: taskExecutor) {
        await actor.performTask("2") // <NSThread: 0x600003a100c0>{number = 2, name = (null)}
    }
}

testDefaultActor()
```


另外验证一下自定义了 sirialExecutor 的 Actor：
```swift
actor MyCustomtActor {
    var count = 0
    func performTask(_ key: String = "") {
        // 确认当前执行在自定义的executor的队列中
        dispatchPrecondition(condition: .onQueue(MyCustomtActor.executor.queue))
        print("performTask \(key) thread: \(Thread.currentThread)")
        count += 1
    }

    static let executor = MySerialExecutor(queue: DispatchQueue.main)

    nonisolated var unownedExecutor: UnownedSerialExecutor {
        MyCustomtActor.executor.asUnownedSerialExecutor()
    }
}
// 仍然按照上面 Default Actor 的测试代码，结果都会运行在自定义的 executor 队列中
```
**自定义`sirialExecutor`的`actor`的方法都会运行在该`executor`或者默认的全局队列中**

如果在`actor`的方法内部创建`Task`，优先级又是如何呢？
```swift
actor MyDefaultActor {
    func performTask() {
        print("performTask thread: \(Thread.currentThread)")
        Task {
            print("task 1 thread: \(Thread.currentThread)")
        }
        let taskExecutor = MyTaskExecutor(queue: DispatchQueue.main)
        Task(executorPreference: taskExecutor) {
            print("task 2 thread: \(Thread.currentThread)")
        }
    }

    nonisolated func testNonisolated() {
        Task {
            print("task 3 thread: \(Thread.currentThread)")
        }
        let taskExecutor = MyTaskExecutor(queue: DispatchQueue.main)
        Task(executorPreference: taskExecutor) {
            print("task 4 thread: \(Thread.currentThread)")
        }
    }
}

func testDefaultActor() async {
    print("testDefaultActor thread: \(Thread.currentThread)")
    let actor = MyActor()
    await actor.performTask()
    actor.testNonisolated()
}

await testDefaultActor()

// 打印结果：
// testDefaultActor thread: <_NSMainThread: 0x6000021d4640>{number = 1, name = main}
// performTask thread: <NSThread: 0x6000021d4b00>{number = 2, name = (null)}
// task 1 thread: <NSThread: 0x6000021c4000>{number = 3, name = (null)}
// task 2 thread: <_NSMainThread: 0x6000021d4640>{number = 1, name = main}
// task 3 thread: <NSThread: 0x6000021c4000>{number = 3, name = (null)}
// task 4 thread: <_NSMainThread: 0x6000021d4640>{number = 1, name = main}
```
在 actor 类型中，Task 设置了 executorPreference 的，就会通过该 executor 执行，否则在全局并发队列运行。

#### `async`函数
```swift
// 没有配置 Global Actor
func asyncFunc(_ i: String) async {
    print("asyncFunc \(i): \(Thread.currentThread)")
}

// 配置了 Global Actor
@MainActor
func globalActorAsyncFunc(_ i: String) async {
    print("globalActorAsyncFunc \(i): \(Thread.currentThread)")
}

func testAsyncFunc() {
    Task {
        await asyncFunc("1") // <NSThread: 0x600003a480c0>{number = 2, name = (null)}
        await globalActorAsyncFunc("1") // <_NSMainThread: 0x6000019e0180>{number = 1, name = main}
    }

    Task { @MainActor in
        await asyncFunc("2") // <NSThread: 0x600003a480c0>{number = 2, name = (null)}
        await globalActorAsyncFunc("2") // <_NSMainThread: 0x6000019e0180>{number = 1, name = main}
    }

    let taskExecutor = MyTaskExecutor(queue: DispatchQueue.main)
    Task(executorPreference: taskExecutor) {
        await asyncFunc("3") // <_NSMainThread: 0x600003a40640>{number = 1, name = main}
        await globalActorAsyncFunc("3") // <_NSMainThread: 0x6000019e0180>{number = 1, name = main}
    }
}

testAsyncFunc()
```
`async`函数自身配置的`Global Actor`优先级最高，`executorPreference`次之。

> 在自定义的`actor`类型的方法中调用`async`函数，会先考虑`async`函数配置的`Global Actor`，如果没有配置，则为`DispatchGlobalTaskExecutor`，不会受`actor`类型是否自定义`serialExecutor`的影响

> 在Swift 5.7实施的[0338-clarify-execution-non-actor-async](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0338-clarify-execution-non-actor-async.md)提案中，说明了**非**actor隔离的`async`函数会在`generic executor`执行，不会在`actor`（是普通的 actor 类型，不是global actor）的executor执行。

#### 线程调度优先级总结
1. Task的代码：`executorPreference` > `global actor` > `default（DispatchGlobalTaskExecutor）`
2. actor的代码：`自定义SerialExecutor` > `Default Actor（executorPreference（上面发现并发调用重叠问题存疑🤨❓） > DispatchGlobalTaskExecutor）`
3. async函数：函数自己配置的`global actor` > `executorPreference` > `default（DispatchGlobalTaskExecutor）`

> `global actor`表示自定义的`GlobalActor`，不是直接定义的`actor`类型

> Swift文档对线程调度和优先级的介绍很少，并且实际测试也有可疑点，所以感觉不记也没事，只需要对用法了解，开发的时候，结合实际验证即可。

## 简要源码分析
### `Task`创建和执行
`Task`的`init`函数是动态生成的，源码可见：[Task+init.swift.gyb](https://github.com/swiftlang/swift/blob/main/stdlib/public/Concurrency/Task%2Binit.swift.gyb)

以 Swift 6.1 为例（6.2增加了name参数），忽略各种判断条件，简化后的代码如下：
```swift
@discardableResult
public init(priority: TaskPriority? = nil, operation: sending @escaping @isolated(any) () async -> Success) {
    // Set up the job flags for a new task.
    let flags = taskCreateFlags(
      priority: priority,
      isChildTask: false,
      copyTaskLocals: ${'true' if not IS_DETACHED else 'false /* detached */'},
      inheritContext: ${'true' if not IS_DETACHED else 'false /* detached */'},
      enqueueJob: true,
      addPendingGroupTaskUnconditionally: false,
      isDiscardingTask: false,
      isSynchronousStart: false)

    let builtinSerialExecutor =
      unsafe Builtin.extractFunctionIsolation(operation)?.unownedExecutor.executor
    var task: Builtin.NativeObject?

    if task == nil {
      // either no task name was set, or names are unsupported
      task = Builtin.createTask(
      flags: flags,
      initialSerialExecutor: builtinSerialExecutor,
      operation: operation).0
    }
    self._task = task!
}

@discardableResult
init(
    executorPreference taskExecutor: consuming (any TaskExecutor)?,
    priority: TaskPriority? = nil,
    operation: sending @escaping () async -> Success
) {
    // Set up the job flags for a new task.
    let flags = taskCreateFlags(
      priority: priority,
      isChildTask: false,
      copyTaskLocals: ${'true' if not IS_DETACHED else 'false /* detached */'},
      inheritContext: ${'true' if not IS_DETACHED else 'false /* detached */'},
      enqueueJob: true,
      addPendingGroupTaskUnconditionally: false,
      isDiscardingTask: false,
      isSynchronousStart: false)
    
    var task: Builtin.NativeObject?
    if task == nil {
      assert(name == nil)
      task = Builtin.createTask(
        flags: flags,
        initialTaskExecutorConsuming: taskExecutor,
        operation: operation).0
    }

    if task == nil {
      // either no task name was set, or names are unsupported
      task = Builtin.createTask(
      flags: flags,
      operation: operation).0
    }
    self._task = task!
}
```
核心就是调用了`Builtin.createTask`在 Runtime 层继续执行。

> `TaskGroup.addTask`和`async let`创建 child task，也是通过`Builtin.createTask`创建的（`TaskGroup`还可能使用`Builtin.createDiscardingTask`）。

以`Builtin.createTask`为例，追踪它的调用流程：
1. `swift_task_create`（Task.cpp）
2. `swift_task_create_commonImpl`（Task.cpp）：记录和获取executorPreference、serialExecutor、parent task等相关信息，并创建`AsyncTask`，
3. `AsyncTask.flagAsAndEnqueueOnExecutor`（TaskPrivate.h）：更新状态，调用`swift_task_enqueue`准备调度
4. `swift_task_enqueueImpl`（Actor.cpp）：根据serialExecutor、executorPreference信息，开始调度
5. 根据情况，调用不同方法：
    * 5.1 `_swift_task_enqueueOnTaskExecutor`：使用了 executorPreference 的情况
    * 5.2 `swift_task_enqueueGlobal`：没有使用 executorPreference ，也没有配置 global actor 的情况。可以继续在源码DispatchGlobalExecutor.cpp（6.0版本）或者ExecutorImpl.swift（6.2版本）中查看`swift_task_enqueueGlobalImpl`
    * 5.3 `swift_defaultActor_enqueue`：默认 actor 类型的情况，会把 job 交给`DefaultActorImpl.enqueue`执行（它会保证串行执行）
    * 5.4 `_swift_task_enqueueOnExecutor`：配置了 global actor 或者自定义 executor 的 actor 的情况
6. 以`_swift_task_enqueueOnTaskExecutor`为例，它会调用`TaskExecutor.enqueue`（TaskExecutor.swift），从而执行到我们自定义的executor中。
7. 在`TaskExecutor.enqueue`中，我们会调用`job.runSynchronously`，以`runSynchronously(on executor: UnownedTaskExecutor)`为例
8. 会执行`_swiftJobRunOnTaskExecutor`（PartialAsyncTask.swift）为例，会执行到`swift_job_run_on_task_executor`（Actor.cpp）
9. 执行到`swift_job_run_on_serial_and_task_executorImpl`（Actor.cpp）
10. 执行`runJobInEstablishedExecutorContext`（Actor.cpp）
11. 执行`runInFullyEstablishedContext`或者`runSimpleInFullyEstablishedContext`（Task.h），分别会执行`ResumeTask`或者`RunJob`，它们都是函数指针。而`ResumeTask`和`RunJob`都是C++的`Job`类构造传入的，并且`AsyncTask`就是`Job`的子类，也就是在构造`AsyncTask`的时候传入。

### runSynchronously
在使用`executorPreference`的情况下，Runtime可以知道当前Task使用了某个`TaskExecutor`，为什么在调用`runSynchronously`方法时，还需要再次传入`executor`？官方文档只说用于上下文信息，但没有更具体的说明，通过源码分析，猜测有如下原因：
1. 比如在`Kotlin`中，`Dispatcher`是开发者自定义，并且可以得到`Runnable`完全自己调度，所以`kotlin`可以实现`Dispatcher1`转发给`Dispatcher2`，让`Dispatcher2`来最终调度，但`context`中仍然是`Dispatcher1`，所以在多次切换`Dispatcher`时会影响协程调度机制的判断，造成多余的调度。而`Swift`的`job.runSynchronously`可以再传入`executor`，开发者可以更准确地告诉`runtime`最终是在哪个`executorr`执行的。比如在`Swift`的Runtime层中，就会对比`executorPrefrence`和当前`job.runSynchronously`传入的`executor`，如果一致，则直接执行，否则会进行调度。
2. 用于提供协程任务的信息，比如`Executor.swift`文件中的`Task.currentExecutor`方法，就会结合`runSynchronously`传入的信息和`executroPreference`综合判断。

### DefaultActor
没有自定义`unownedExecutor`的`actor`类型，会作为`DefaultActor`类型，在`SILGenConstructor.cpp`的`emitClassConstructorInitializer`方法中可以看到，调用`emitDefaultActorInitialization`，也就会在`GenBuiltin.cpp`文件的`InitializeDefaultActor`情况调用`getDefaultActorInitializeFunctionPointer()`对应函数，在`RuntimeFunctions.def`文件中则可以看到是`swift_defaultActor_initialize`。

然后我们可以通过默认`actor`类型编译后生成的ir文件中验证，在其中可以看到`swift_defaultActor_initialize`函数的调用，在生成的actor class的init方法中调用用于初始化`DefaultActor`。

> 默认actor会通过`swift_defaultActor_enqueue`方法调度任务，实际就是交给`DefaultActorImpl::enqueue`执行

> 在 Runtime 中会看到`SerialExecutorRef`的`isGeneric()`方法和`isDefaultActor()`的判断，根据源码推断，`isGeneric()`表示没有绑定到任何具体的 executor 或 actor。当 actor 类型自定义了 unownedExecutor，其对应的 SerialExecutorRef 不满足 isGeneric()。

### Task.defaultExecutor
在`Executor.swift`中为`Task`添加了`defaultExecutor`属性，通过`_createDefaultExecutors`方法获取，最终通过`PlatformExecutorFactory`的具体实现提供，可以查看`PlatformExecutorDarwin.swift`中的实现。

`globalConcurrentExecutor`的实际实现也就是`Task.defaultExecutor`，比如实现为`DispatchGlobalTaskExecutor`，它调度任务的流程如下：
1. `_enqueueJobGlobal`
2. `Task.defaultExecutor.enqueue(unownedJob)`
3. `_dispatchEnqueueGlobal`
4. `dispatchEnqueue`（DispatchGlobalExecutor.cpp）
5. 执行到 dispatch_async_f（Concurrency-libdispatch.cpp） 或者 dispatch_async_swift_job

更加底层的实现则为`libdispatch`库，可以查看源码:https://github.com/swiftlang/swift-corelibs-libdispatch，https://github.com/apple-oss-distributions/libdispatch

### 总览
通过对`Swift`并发的源码阅读，可以得知`Swift`并发的实现是在 C++ Runtime 层。`Task.init`在底层会创建一个`AsyncTask`对象（`Job`的子类）。`AsyncTask`是运行时中的原生任务对象，负责实际的任务调度和执行。当我们用`async let`、`Task.init`、`actor`等代码时，`Swift`编译器会自动帮我们完成代码转化。`AsyncTask`只能通过编译器和运行时的特殊接口操作，我们在`Swift`层无法直接访问其内部结构，也无法操作`AsyncTask`的创建、排队、分发等细节。

对于想了解的细节问题，都可以通过看`Swift`源码、提案来了解，源码中的test相关代码可以用来了解使用方式，另外，可以通过`swiftc -emit-irgen`、`swiftc -emit-bc`等命令查看编译后的代码流程。

> 比如想了解异步函数编译后的代码，则可以查看Swift的`GenFunc.cpp`源码中的`createAsyncSuspendFn()`方法。

## 其他常用
### Task.immediate
`Task`立即执行的版本，不需要将新任务排入队列并“稍后运行”，而是直接在当前调用上下文立即运行任务体，直到第一个await点才开始调度）。
```swift
Task.immediate {}
```
> https://github.com/swiftlang/swift-evolution/blob/main/proposals/0472-task-start-synchronously-on-caller-context.md

### TaskLocal
`TaskLocal`是和`Task`关联的局部数据，类似线程中使用的`ThreadLocal`。

使用方式如下示例：
```swift
// TaskLocal 必须是静态属性或者全局属性：
enum Example {
    @TaskLocal
    static var traceID: TraceID?
}

// 全局的 task local 属性从 Swift 6.0 开始支持
@TaskLocal
var contextualNumber: Int = 12
```

child task可以继承父级的`TaskLocal`值（Task.detached 不行）
```swift
class Example {
    @TaskLocal
    static var data: String = "default"
}

func taskLocalTest() {
    print("taskLocalTest \(Example.data)")
}

func taskLocalAsyncTest() async {
    print("taskLocalAsyncTest \(Example.data)")
}

Task {
    await Example.$data.withValue("111111") {
        print("in1 withValue \(Example.data)") // 打印：in1 withValue 111111
        Task {
            print("inner child task: \(Example.data)") // 打印：inner child task: 111111
        }
        await taskLocalAsyncTest() // 打印：taskLocalAsyncTest 111111
        taskLocalTest() // 打印：taskLocalTest 111111
        
        Task.detached {
            print("child detached task \(Example.data)") // 打印：child detached task default
        }

        await Example.$data.withValue("22222") {
            print("in2 withValue \(Example.data)") // 打印：in2 withValue 22222
        }
    }
    print("task: \(Example.data)") // 打印：task: default
    Task {
        print("outside child task: \(Example.data)") // 打印：outside child task: default
    }
}
```
修改`TaskLocal`的值，需要通过`withValue`方法，并且修改的值仅在`withValue`函数的闭包内生效，也就是说这是临时修改。
因为`withValue`是在属性包装器类型上声明的，因此为了访问此函数，必须在属性名称后加上 `$` 符号，以访问属性包装器的投影值而不是包装值本身：

> https://github.com/swiftlang/swift-evolution/blob/main/proposals/0311-task-locals.md

### @concurrent
Swift 6.2 的[SE-0461 提案](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0461-async-function-isolation.md)引入了`@concurrent`和`nonisolated(nonsending)`等用法，用于控制非隔离异步函数的执行上下文。当前正式版本的Xcode还只能使用 Swift 6.1，不方便验证，所以暂且不分析。

### @isolated(any)
Swift 6.0 在 [提案0431-isolated-any-functions](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0431-isolated-any-functions.md)中添加了`@isolated(any)`函数类型。

背景：比如传入`Task.init`的函数，由于不知道该函数的隔离（isolation）信息，只能从全局并发Executor中开始执行，执行到函数内的挂起点时再调度到具体的Executor中，存在多余的调度。并且，由于先在全局并发Executor中执行，可能导致并发的多个任务的顺序不一定与创建任务的顺序匹配。

解决这些问题的方式就是增加一个可以携带隔离信息的函数类型，比如`Task.init`之类用于创建任务的API都更新为接收`@isolated(any)`函数。

例如：
```swift
func doSomething(_ myClosure: @isolated(any) () async -> Void) async {
    print("myClosure isolation:", myClosure.isolation)

    await myClosure()
}
```
`myClosure`的类型为`@isolated(any)`函数，所以可以获取该闭包的隔离信息。如果是非隔离函数，则isolation为`nil`；如果是隔离到`global actor`的函数，则isolation为该`GlobalActor.shared`；如果是隔离到特定的`actor`对象，则isolation为该`actor`对象。

`@isolated(any)`是一种函数类型修饰符，表示函数的隔离状态是动态的，可以在运行时绑定到任意 actor 的隔离域，或者是无隔离的（non-isolated）。在官方提案中，描述不太容易理解，简单来说，函数内部可以得到闭包的隔离信息，则可以做出更智能的调度决策。

参考：
* https://nshipster.com/isolated-any/
* https://jano.dev/swift/2025/08/05/isolated-any.html
* https://eunjin3786.tistory.com/672

### #isolation
[提案0420](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0420-inheritance-of-actor-isolation.md)中在 Swift 6.0 增加了`#isolation`宏，用于获取当前的`isolation`，如果是`nil`则表示非隔离。

```swift
// 打印当前的 isolation
let currentIsolation = #isolation
print("current isolation: \(currentIsolation)")

// 作为函数的参数默认值
func stopUpdates(_ isolation: isolated (any Actor)? = #isolation) {
    
}

// 或者
func stopUpdates(_ isolation: isolated (Actor)? = #isolation) {
    
}
```

### @preconcurrency
`@preconcurrency` 是 Swift 中的一个属性，用于将类型、函数或声明标记为来自并发前的代码库，这意味着它是在 Swift 的并发模型（在 Swift 5.5 中引入）可用之前编写的。此属性可帮助 Swift 编译器放宽一些更严格的并发检查，以实现向后兼容性。当您使用 `@preconcurrency` 时，您实际上是在告诉编译器，您正在使用的代码或 API 可能不符合 Swift 的新并发规则，但在并发上下文中使用仍然是安全的。

比如`DispatchQueue.asyncAfter`函数就使用了`@preconcurrency`：
```swift
@preconcurrency public func asyncAfter(deadline: DispatchTime, qos: DispatchQoS = .unspecified, flags: DispatchWorkItemFlags = [], execute work: @escaping @Sendable @convention(block) () -> Void)
```

## 异步序列
`AsyncSequence`协议表示一个异步、顺序、可遍历的类型，其实就是类似`Kotlin`的`Flow`、`Python`的`异步生成器`。

实现此协议的重点就是提供一个`AsyncIterator`，用于对外提供异步序列数据。
```swift
struct Counter : AsyncSequence {
  let howHigh: Int

  struct AsyncIterator : AsyncIteratorProtocol {
    let howHigh: Int
    var current = 1
    mutating func next() async -> Int? {
      // We could use the `Task` API to check for cancellation here and return early.
      guard current <= howHigh else {
        return nil
      }

      let result = current
      current += 1
      return result
    }
  }

  func makeAsyncIterator() -> AsyncIterator {
    return AsyncIterator(howHigh: howHigh)
  }
}

for await i in Counter(howHigh: 3) {
  print(i)
}

/* 
Prints the following, and finishes the loop:
1
2
3
*/
```

但这样使用还是有点麻烦，所以Swift还提供了实现`AsyncSequence`协议的`AsyncStream`，以及可抛异常的`AsyncThrowingStream`。通过它们可以更方便地创建异步序列。

比如通过`AsyncStream.init`可以创建一个异步流（相当于冷流）：
```swift
let asyncStream = AsyncStream<Int> { continuation in
    Task.detached {
        for _ in 0..<10 {
            try? await Task.sleep(nanoseconds: 1_000_000_000)
            continuation.yield(Int.random(in: 1...10))
        }
        continuation.finish()
    }
}

for try await number in asyncStream {
    print(number)
}
// 将每秒打印一个1到10的随机数
```

也可以使用`AsyncStream.makeStream`方法实现热流（验证是单播，并非广播）：
```swift
let (stream, continuation) = AsyncStream.makeStream(of: Int.self)
// 数据产生端
Task {
    for i in 1...5 {
        continuation.yield(i)
        try await Task.sleep(nanoseconds: 1_000_000_000)
    }
    continuation.finish()
}

// 消费者接收数据
Task {
    for await value in stream {
        print("消费者收到: \(value)")
    }
}
```

`AsyncStream`没有支持多播，可以查看社区讨论：https://forums.swift.org/t/consuming-an-asyncstream-from-multiple-tasks/54453

检测取消：

> 如果`AsyncStream`需要在结束时释放某些资源，可以设置`continuation.onTermination`回调，以免内存和资源泄露

相关提案：
* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0298-asyncsequence.md
* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0314-async-stream.md
* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0388-async-stream-factory.md
* https://github.com/swiftlang/swift-evolution/blob/main/proposals/0406-async-stream-backpressure.md

# Combine
`Combine`是Swift官方提供的响应式编程框架。