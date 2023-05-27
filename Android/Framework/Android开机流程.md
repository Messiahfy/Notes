#### C++ Framework（Native层）
启动 Linux 内核后，会启动 `Native` 层第一个进程 `init`（pid=1），`init` 进程是所有用户进程的鼻祖。`init` 进程会解析 `init.rc` 文件来陆续启动其他关键的系统服务进程，如 `servicemanager`、`Zygote`等。

#### Native和Java的交接点：Zygote进程
> /frameworks/base/cmds/app_process/App_main.cpp
> /frameworks/base/core/jni/AndroidRuntime.cpp

`Zygote是所有Java进程的父进程。`

启动 `Zygote`实际是执行 App_main.cpp --> AndroidRuntime.cpp，其中会创建Java虚拟机、注册JNI方法，最后反射调用 `ZygoteInit.main()` 方法，从而进入 `Java` 层。

#### Java Framework层
Zygote执行到Java层：
* 注册Socket
* 预加载虚拟机运行的各类资源
* 启动 System Server


System Server进程：
* 启动Binder线程池，可以与其他进程通信
* 启动和管理ActivityManager，WindowManager，PackageManager等。

| 类型 | 服务 |
| --- | --- |
| Bootstrap | ActivityManagerService<br/>PackageManagerService<br/>PowerManagerService 等 |
| Core | BatteryService 等|
| Other | InputManagerService<br/>WindowManagerService<br/>NotificationManager 等|

#### Launcher
AMS服务启动Launcher