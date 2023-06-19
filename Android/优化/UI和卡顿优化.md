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

读取 /sys/devices 中的 time_in_state

通过轮询，前台30秒 后台10分钟 超过临界值，就缩短轮询间隔，缩短到30秒内， 每秒都检测  还是超过 就上报异常线程(本进程内cpu占用超过5%)堆栈，没有超过 就等一会儿再开始轮询。根据线程堆栈和cpu使用率 结合。有些场景可以忽略，比如手机温度高、电量低、节能模式等情况

### 帧率和丢帧次数
只考虑帧率，不能完全表现卡顿的情况，比如帧率达到55，连续丢了5帧，还是会有卡顿感（可以把统计时间减少，比如本来1s，改为100ms区间来统计）。

## 线下卡顿排查工具
前面是命令的方式，还有图形化的简单方式。

### 开发者模式
HWUI呈现模式

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

1. 布局优化：耗时在于加载xml和反射。减少嵌套，merge，避免过度绘制，异步加载（用的少），xml转代码（部分属性不能用代码设置，编译时间长），compose
2. 卡顿监控：消息执行、Vsync回调

## 监控
卡顿监控：Looper.setMessageLogging，监控每个消息的执行时间。
FPS监控：Choreographer.postFrameCallback。不能直接使用两次回调的时间差，例如一次回调在第一帧的开始，二次回调在第二帧的末尾，最大间隔可以接近32ms，所以还应该通过反射获取Choreographer.mFrameInfo里面的VSYNC等时间数据，综合计算。还要考虑触摸事件以及开始滑动（ViewTreeObserver.OnScrollChangedListener ）等因素，比如页面静止时其实没必要计算帧率。Window.OnFrameMetricsAvailableListener也可以读取帧耗时信息
部分插桩：线上可以对部分函数插桩，监控耗时函数。比如锁、循环、IO操作、绘制、方法体较大之类的函数。配合获取堆栈

在子线程高频获取主线程的堆栈，重复的堆栈信息很可能就是耗时调用

使用systrace，但要字节码插桩（ASM+transform），不然手动去每个函数添加trace代码不现实，但要避免开始和结束的trace方法没有成对调用，可利用try-catch 

cpu监控，可以例如30s抓一次 /proc/%s/status等相关数据

## ui绘制流程
加载xml、测量、布局、绘制

## 方式
* 动画：降低原生动画频率、并且低端机可以降级（去掉动画）、绘制泄漏
* 缩小刷新区域、clipRect、invalidate，开发者选项 -> 显示GPU视图更新
* 过度绘制、移除无用背景
* 层级过深
* 按需加载、include、merge、viewStub
* inflate：x2c
* 去掉debug日志
* 消息重排、分散
* Java锁
* IO调用、binder调用、系统方法调用
* 高频函数调用：单次时间短，多次也会累积较长时间
* 缓存