anr会在 /data/anr 目录下面生成 traces.txt 文件。

### ANR 监控
* 在运行期间FileObserver监听 /data/anr/ 目录下的否有新增".tarce"结尾文件来收集 ANR 事件，需要对包名时间等进行过滤。**这个方式仅作思路参考，Android 5 以上第三方app无权限**
* 开启一个子线程 ANRWatchDog，每隔段时间往主线程发一个消息，然后去检查消息是否被执行
* 监听 SIGNALQUIT 信号
* Looper的setMessageLogging，消息执行前后间隔时间，可以缓存10S以上的数据，先进先出，达到保存ANR前10秒以上消息调度情况的目的

### 获取ANR日志
ActivityManager获取ProcessErrorStateInfo，如果包含NOT_RESPONDING 状态的消息，则认为发生了 ANR。还有其他错误信息。

### 结合调用栈
因为发生anr时的调用栈，并不一定就是导致anr的原因，所以可以使用Looper的setMessageLogging的方式得到ANR前一段时间的消息调度情况，并且结合间隔采样抓函数栈，对比不同时间的函数栈，得到造成卡顿的原因。采样抓函数栈。

Systrace和Debug类性能不好，仅适合线下使用。所以线上采样的方式还需要自行开发，例如Thread.getStackTrace内部使用了StackVisitor::WalkStack，可以对其做一些精简来使用。

[ANR 优化实践系列 - 监控工具与分析思路](https://juejin.cn/post/6942665216781975582)