anr会在 /data/anr 目录下面生成 traces.txt 文件。

### 可能原因
* 调度消息存在耗时操作
* 主线程被其他线程阻塞
* CPU被严重抢占，调度延迟
* IO
* 系统内存紧张
* CPU限频、锁核

### ANR 监控
* 在运行期间FileObserver监听 /data/anr/ 目录下的否有新增".tarce"结尾文件来收集 ANR 事件，需要对包名时间等进行过滤。**这个方式仅作思路参考，Android 5 以上第三方app无权限**
* 开启一个子线程 ANRWatchDog，每隔段时间往主线程发一个消息，然后去检查消息是否被执行
* 监听 SIGNALQUIT 信号，并检测主线程Looper中消息的when变量和当前时间比较，确定是否阻塞较久
* Looper的setMessageLogging，消息执行前后间隔时间，可以缓存10S以上的数据，先进先出，达到保存ANR前10秒以上消息调度情况的目的

### 获取ANR日志
ActivityManager获取ProcessErrorStateInfo，如果包含NOT_RESPONDING 状态的消息，则认为发生了 ANR。还有其他错误信息。

ANR堆栈不一定刚好是耗时的堆栈，可能耗时的任务执行完还在执行一些不耗时的任务，回复超时的任务还在消息队列更后面，此时还没有回复SystemServer的超时判断

nativePollOnce：一般就是当前消息之前存在历史耗时消息，导致当前消息来不及调度执行；或者高负载。

### 结合调用栈
因为发生anr时的调用栈，并不一定就是导致anr的原因，所以可以使用Looper的setMessageLogging的方式得到ANR前一段时间的消息调度情况，并且结合间隔采样抓函数栈，对比不同时间的函数栈，得到造成卡顿的原因。采样抓函数栈。

Systrace和Debug类性能不好，仅适合线下使用。所以线上采样的方式还需要自行开发，例如Thread.getStackTrace内部使用了StackVisitor::WalkStack，可以对其做一些精简来使用。

[ANR 优化实践系列 - 监控工具与分析思路](https://juejin.cn/post/6942665216781975582)
https://juejin.cn/post/7181731795439157306

### 分析思路
1. log（logcat、内核日志）、cpu、内存
2. CPU占比、Top进程CPU占比
3. 主线程线程状态、业务逻辑堆栈
4. 结合CPU、内存等系统信息和ANR堆栈
5. 业务具体场景分析

日志分析：关注Sys、User、IO占比
* 单个进程占比超过40%说明很有可能异常；内核占比一般10%以下；CPU负载为CPU核数的1.5倍以内较正常。
* mmcqd 进程执行IO读写，kswapd 进程执行内存交换，这两个进程如果超过10%则可任务比较繁忙。