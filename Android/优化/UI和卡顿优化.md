CPU 时间分为用户时间和系统时间。用户时间是执行用户态应用程序代码所消耗的时间;系统时间是执行内核态系统调用所消耗的时间，包括 I/O、锁、中断以及其他系统调用的时间。

## 卡顿分析指标
### CPU使用率
cat /proc/stat 整体CPU时间
cat /proc/[pid]/stat 进程的CPU时间
cat  /proc/[pid]/task/[tid]/stat 线程的CPU时间

打印结果详细含义：http://www.samirchen.com/linux-cpu-performance/

strace 命令可以跟踪某个进程中所有的系统调用  
top 命令查看各个进程实时的资源占用情况

如果 CPU 使用率长期大于 60% ，表示系统处于繁忙状态，需要进一步分析用户时间和系统时间的比例。对于普通应用程序，系统时间不会长期高于 30%，如果超过，就应该进一步检查是 I/O 过多，还是其他的系统调用问题。

### CPU 饱和度
CPU 饱和度反映的是线程排队等待 CPU 的情况，也就是 CPU 的负载情况。

CPU 饱和度首先会跟应用的线程数有关，如果启动的线程过多，容易导致系统不断地切换 执行的线程，把大量的时间浪费在上下文切换，我们知道每一次 CPU 上下文切换都需要刷 新寄存器和计数器，至少需要几十纳秒的时间

1. /proc/[pid]/sched 这里特别需要注意nr_involuntary_switches被动切换的次数。
    * nr_voluntary_switches:
    主动上下文切换次数，因为线程无法获取所需资源导致上下文切换，最普遍的是 IO。
    * nr_involuntary_switches:
    被动上下文切换次数，线程被系统强制调度导致上下文切换，例如大量线程在抢占 CPU。
    * se.statistics.iowait_count:IO 等待的次数
    * se.statistics.iowait_sum: IO 等待的时间

2. 使用/proc/[pid]/schedstat
    * time spent on the cpu
    * time spent waiting on a runqueue
    * 上下文切换次数

3. /proc/loadavg 打印结果前三个数为1、5、15分钟内的平均进程数，第四个数的分子是正在运行的进程数，分母是进程总数，最后一个数是最近运行的进程ID号。

4. cat /proc/[pid]/task 可以看到各个线程，cat /proc/[pid]/task/[tid]/stat 查看线程的CPU使用情况


5. uptime 命令可以检查 CPU 在 1 分钟、5 分钟和 15 分钟内的平均负载。 比如一个 4 核的 CPU，如果当前平均负载是 8，这意味着每个 CPU 上有一个线程在运行，还有一个线程在等待。一般平均负载建议控制在“0.7 × 核数”以内
    * 当前时间
    * 系统运行的时间
    * 当前登录到系统上的用户数
    * 一分钟、五分钟、十五分钟的平均负载

另外一个会影响 CPU 饱和度的是线程优先级，线程优先级会影响 Android 系统的调度策略。在 CPU 繁忙的时候，线程调度会对执行效率有非常大的影响。

## 卡顿排查工具
前面是命令的方式，还有图形化的简单方式。

### Traceview
分析程序执行流程耗时，生成方式如下三种：

1. 可以在代码中使用Debug类的方法来生产trace文件
```
Debug.startMethodTracing("sample"); // 开始 trace
...
Debug.stopMethodTracing();  // 结束 trace
```
2. ～/Library/Android/sdk/tools/monitor 
3. 使用Android Studio的profiler的cpu分析工具。

###  Systrace
谷歌在Android一些关键源码中添加了追踪代码，如果要追踪我们自己的代码可以写如下代码：
```
Trace.beginSection("sample");
...
Trace.endSection();
```

#### 启动方式
执行命令：
```
~/Library/Android/sdk/platform-tools/systrace/systrace.py
```
命令还有各种选项，具体使用方式   https://developer.android.google.cn/topic/performance/tracing/command-line#app-trace

执行后会生成html文件，可以用chrome打开

#### 分析结果
可参考如下网页：
https://developer.android.google.cn/topic/performance/tracing/navigate-report  
https://www.androidperformance.com/2019/07/23/Android-Systrace-Pre/  
https://www.cnblogs.com/1996swg/p/10007602.html  

Android 9(API 28)开始，开发者模式提供System Tracing 应用
https://developer.android.google.cn/topic/performance/tracing/on-device#converting_between_trace_formats

### Simpleperf
根据Android版本，不同程度地支持Native和Java

### Profiler
Android Studio内置工具

-----------------------------

Choreographer  postFrameCallback
Looper  setMessageLogging

## UI优化
### 官方工具
* sdk/tools/bin/UiAutomatorViewer
* adb shell uiautomator dump /data/local/tmp/uidump.xml 然后pull
* Android Studio 内置 layout inspector
### 参考
[检查GPU渲染速度和过度绘制](https://developer.android.google.cn/topic/performance/rendering/inspect-gpu-rendering)
[Android 屏幕绘制机制及硬件加速](https://blog.csdn.net/qian520ao/article/details/81144167)
http://hukai.me/android-performance-render/
https://www.youtube.com/watch?v=zdQRIYOST64
https://mp.weixin.qq.com/s/QoFrdmxdRJG5ETQp5Ua3-A
https://source.android.google.cn/devices/graphics

[Android性能优化之绘制优化](https://juejin.im/post/6844904080989487118)
[深入探索Android布局优化（上）](https://juejin.im/post/6844904047355363341)
[深入探索Android卡顿优化（上）](https://juejin.im/post/6844904062610046990)

[性能优化](https://www.jianshu.com/p/5950e1e8b31e)

1. 布局优化：耗时在于加载xml和反射。减少嵌套，merge，避免过度绘制，异步加载（用的少），xml转代码（部分属性不能用代码设置，编译时间长），compose