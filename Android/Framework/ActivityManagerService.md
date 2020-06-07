Android 10源码
## AMS的启动
SystemServer中调用 `startBootstrapServices`：
```
private void startBootstrapServices() {
    ...
    //启动AMS服务，AMS构造函数会初始化很多东西，比如多个线程、广播接收队列、某些文件目录等。
    mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
    //设置其他
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    ...
    mActivityManagerService.initPowerManagement();
    ...
    //向ServiceManager注册AMS、meminfo、cpuinfo 等服务
    mActivityManagerService.setSystemProcess();

}
```

然后在SystemServer中调用 `startOtherServices`：
```
...
//安装系统provider
mActivityManagerService.installSystemProviders();
...
mActivityManagerService.setWindowManager(wm);
...
//systemReady方法会执行这里传入的runnable，然后执行其他代码
mActivityManagerService.systemReady(() -> {
        mSystemServiceManager.startBootPhase(
                SystemService.PHASE_ACTIVITY_MANAGER_READY);
        try {
            mActivityManagerService.startObservingNativeCrashes();
        } catch (Throwable e) {
            reportWtf("observing native crashes", e);
        }
        //启动WebView
        final String WEBVIEW_PREPARATION = "WebViewFactoryPreparation";
        Future<?> webviewPrep = null;
        if (!mOnlyCore && mWebViewUpdateService != null) {
            webviewPrep = SystemServerInitThreadPool.get().submit(() -> {
                Slog.i(TAG, WEBVIEW_PREPARATION);
                TimingsTraceLog traceLog = new TimingsTraceLog(
                        SYSTEM_SERVER_TIMING_ASYNC_TAG, Trace.TRACE_TAG_SYSTEM_SERVER);
                traceLog.traceBegin(WEBVIEW_PREPARATION);
                ConcurrentUtils.waitForFutureNoInterrupt(mZygotePreload, "Zygotepreload");
                mZygotePreload = null;
                mWebViewUpdateService.prepareWebViewInSystemServer();
                traceLog.traceEnd();
            }, WEBVIEW_PREPARATION);
        }
        ...
        //以下代码可能省略try-catch或者判空语句

        //启动系统UI
        startSystemUi(context, windowManagerF);
        ...

        //调用各种服务的systemReady
        networkManagementF.systemReady();
        CountDownLatch networkPolicyInitReadySignal = null;
        if (networkPolicyF != null) {
            networkPolicyInitReadySignal = networkPolicyF
                    .networkScoreAndNetworkManagementServiceReady();
        }
        ipSecServiceF.systemReady();
        networkStatsF.systemReady();
        connectivityF.systemReady();
        networkPolicyF.systemReady(networkPolicyInitReadySignal);
        // Wait for all packages to be prepared
        mPackageManagerService.waitForAppDataPrepared();
        // 确认 webview 启动完成
        if (webviewPrep != null) {
            ConcurrentUtils.waitForFutureNoInterrupt(webviewPrep, WEBVIEW_PREPARATION);
        }
        mSystemServiceManager.startBootPhase(
                SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
        traceBeginAndSlog("StartNetworkStack");
        try {
            // Note : the network stack is creating on-demand objects that need to send
            // broadcasts, which means it currently depends on being started after
            // ActivityManagerService.mSystemReady and ActivityManagerServicemProcessesReady
            // are set to true. Be careful if moving this to a different place in the
            // startup sequence.
            NetworkStackClient.getInstance().start();
        } catch (Throwable e) {
            reportWtf("starting Network Stack", e);
        }
        traceEnd();
        traceBeginAndSlog("StartTethering");
        try {
            // TODO: hide implementation details, b/146312721.
            ConnectivityModuleConnector.getInstance().startModuleService(
                    TETHERING_CONNECTOR_CLASS,
                    PERMISSION_MAINLINE_NETWORK_STACK, service -> {
                        ServiceManager.addService(Context.TETHERING_SERVICE, service,
                                false /* allowIsolated */,
                                DUMP_FLAG_PRIORITY_HIGH | DUMP_FLAG_PRIORITY_NORMAL);
                    });
        } catch (Throwable e) {
            reportWtf("starting Tethering", e);
        }
        traceEnd();

        //调用各种服务的systemRunning
        locationF.systemRunning();
        countryDetectorF.systemRunning();
        networkTimeUpdaterF.systemRunning();
        inputManagerF.systemRunning();
        telephonyRegistryF.systemRunning();
        mediaRouterF.systemRunning();
        mmsServiceF.systemRunning();
        try {
            // TODO: Switch from checkService to getService once it's always
            // in the build and should reliably be there.
            final IIncidentManager incident = IIncidentManager.Stub.asInterface(
                    ServiceManager.getService(Context.INCIDENT_SERVICE));
            if (incident != null) {
                incident.systemRunning();
            }
        } catch (Throwable e) {
            reportWtf("Notifying incident daemon running", e);
        }
        traceEnd();
    }, BOOT_TIMINGS_TRACE_LOG);
```

AMS的systemReady中的其他代码，会执行：启动 Persistent 应用，启动 home Activity，恢复栈顶Activity等任务。

## AMS中的Activity管理
> 这部分的代码，例如ActivityStackSupervisor、ActivityDisplay等文件各版本都在调整。

* ActivityRecord：是Activity在AMS中的对应记录，由于Activity实际由AMS管理，所以很多信息是存储在ActivityRecord中的
* TaskRecord：ActivityRecord所属的任务栈，其中有taskId等信息，并且持有当前栈中的ActivityRecord列表mActivities；也持有它所属的ActivityStack
* ActivityStack：ActivityRecord和TaskRecord的管理者。
* ActivityStackSupervisor：管理ActivityStack等，但10版本代码，已经将一些管理代码放到了RootActivityContainer、ActivityDisplay中
* ActivityDisplay：表示一个屏幕，例如主屏幕、外接屏幕。10版本，几种ActivityStack都在此类中直接管理

以上类并非完成层次分离，例如ActivityRecord、ActivityStack等也会直接持有ActivityStackSupervisor来调用其操作。

启动Activity，经过Instrumentation --> ActivityTaskManager --> ActivityTaskManagerService --> ActivityStarter --> RootActivityContainer --> ActivityStack


ActivityStarter中会执行到 startActivityUnchecked，然后mRootActivityContainer.resumeFocusedStacksTopActivities，然后 ActivityStack.resumeTopActivityUncheckedLocked -> resumeTopActivityInnerLocked

* 让当前Activity暂停：startPausingLocked，最终会跨进程调用到App的ActivityThread
* 创建新的Activity：StackSupervisor.startSpecificActivityLocked，如果进程已存在，则直接 realStartActivityLocked ，其中 ATMS中的ClientLifecycleManager.scheduleTransaction会推动其生命周期；而如果进程不存在，则会先创建进程，然后创建对应的运行环境，例如绑定到AMS等，然后回调Applocation的onCreate，然后AMS调用到 realStartActivityLocked 推动Activity的生命周期，App中的Activity在ActivityThread中创建。

https://blog.csdn.net/nanyou519/article/details/104735722

## AMS中的Service管理
* ActiveServices：用于 ActivityStackSupervisor 管理Service
* ServiceRecord：是Service在AMS的记录
* IntentBindRecord：含于ServiceRecord，表示一个绑定到Service的Intent。其中记录了 ArrayMap<ProcessRecord, AppBindRecord>，一个进程对应一个AppBindRecord
* AppBindRecord：Service和它的某一个客户端应用的关联记录，其中有 ConnectionRecord 集合
* ConnectionRecord：描述一个对Service的单次绑定

#### StartService简要流程：
ContextWrapper --> ContextImpl --> AMS --> ActiveServices#startServiceLock  --> ActiveServices#startServiceInnerLocked --> ActiveServices#bringUpServiceLocked

到了bringUpServiceLocked
* 如果进程已启动，则直接执行 realStartServiceLocked，其中 1.调用ActivityThread的scheduleCreateService，使用AMS管理Service生命周期onCreate 2.sendServiceArgsLocked --> 跨进程 ActivityThread.scheduleServiceArgs会执行到onStartCommand
* 否则，先启动进程，并把要启动的Service放到ActiveService的mPendingServices中，启动进程后，会调用AMS#attachApplicationLocked -->
 ActiveServices#attachApplicationLocked，这里就会对mPendingServices中的每个待启动的ServiceRecord调用realStartServiceLocked

#### StopService简要流程：
ContextWrapper --> ContextImpl  --> AMS --> ActiveServices#stopServiceLocked --> ActiveServices#bringDownServiceIfNeededLocked --> ActiveServices#bringDownServiceLocked  --> ActivityThread#scheduleStopService --> ActivityThread#handleStopService 这里会调用Service的onDestroy --> AMS#serviceDoneExecuting

-----------------------------------------

#### bindService简要流程：
ContextWrapper --> ContextImpl --> AMS#bindIsolatedService --> ActiveServices#bindServiceLocked --> bringUpServiceLocked

* ContextImpl的bindServiceCommon会把ServiceConnection包装到LoadedApk.ServiceDispatcher
* bindServiceLocked中会设置ServiceRecord的bindings，还会调用ServiceRecord的addConnection，把ServiceConnection的包装对象放进去
* 这里和startService一样执行到了bringUpServiceLocked，且同样会执行到 realStartServiceLocked，进行和startService一样的流程，还会执行 requestServiceBindingsLocked，这里面会读取设置的bindings

ActiveServices#requestServiceBindingsLocked --> requestServiceBindingLocked --> 跨进程 ActivityThread#scheduleBindService --> handleBindService 这里onBind会返回IBinder --> AMS#serviceDoneExecuting

onBind返回IBinder --> AMS#publishService --> ActiveServices#publishServiceLocked --> c.conn.connected 这里再跨进程回App --> LoadedApk.ServiceDispatcher.InnerConnection#connected --> ServiceDispatcher#connected --> RunConnection#run --> ServiceConnection#onServiceConnected

#### unBindService简要流程
ContextWrapper --> ContextImpl --> AMS#unbindService --> ActiveServices#unbindServiceLocked --> removeConnectionLocked --> 跨进程 ActivityThread#scheduleUnbindService --> handleUnbindService 这里会onUnBind --> AMS


## 广播
普通广播为并行广播，有序广播为串行广播，黏性广播为串行广播
#### 广播的注册 registerReceiver
ContextWrapper --> ContextImpl#registerReceiver --> registerReceiverInternal --> AMS#registerReceiver

* ContextImpl#registerReceiver中会把 BroadcastReceiver 封装到LoadedApk.ReceiverDispatcher中，然后将LoadedApk.ReceiverDispatcher的mIIntentReceiver作为receiver传到AMS
* AMS#registerReceiver中会把receiver放到RegisteredReceivers中
* 并创建BroadcastFilter放到 `mReceiverResolver`（实际就是动态注册：在发送广播时会在其中查询），且将BroadcastFilter放到BroadcastRecor然后放到mParallelBroadcasts

#### 广播的发送
ContextWrapper --> ContextImpl --> AMS#broadcastIntent --> AMS#broadcastIntentLocked

broadcastIntentLocked方法很长
1. 设置flag
2. 权限检查
3. 根据Action，处理系统广播
4. 添加Sticky广播到AMS的mStickyBroadcasts
5. 查询广播接收器：receivers记录匹配的所有静态广播接收器，registeredReceivers记录动态注册的广播接收器
6. 处理并行广播：receivers放到BroadcastRecord，然后传到enqueueParallelBroadcastLocked方法，放到mParallelBroadcasts列表中，然后执行scheduleBroadcastsLocked
7. 合并registeredReceivers到receivers
8. 处理串行广播：enqueueOrderedBroadcastLocked，scheduleBroadcastsLocked

#### 广播处理
BroadcastQueue#scheduleBroadcastsLocked --> processNextBroadcast --> processNextBroadcastLocked

在processNextBroadcastLocked中：
1. 先处理并行广播 deliverToRegisteredReceiverLocked --> performReceiveLocked 这里不论是跨进程到ActivityThread#scheduleRegisteredReceiver还是直接都会执行 LoadedApk.ReceiverDispatcher.InnerReceiver#performReceive --> ReceiverDispatcher.Args.getRunnable 这里就会执行 onReceive

静态注册的receivers始终采用串行方式处理（processNextBroadcast），而动态注册的registeredReceivers处理方式是串行还是并行方式, 取决于广播的发送方式(processNextBroadcast)