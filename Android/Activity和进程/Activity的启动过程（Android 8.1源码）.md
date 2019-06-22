[死磕Android_View工作原理你需要知道的一切](https://blog.csdn.net/xfhy_/article/details/90270630)
## 1.概述
&emsp;&emsp;`Activity`的启动过程分为两种，一种是根`Activity`的启动过程，另一种是普通`Activity`的启动过程，根`Activity`指的是应用程序启动的第一个`Activity`，因此根`Activity`的启动过程一般情况下也可以理解为应用程序的启动过程（包含了进程的创建）。普通`Activity`指的是除了应用程序启动的第一个`Activity`之外的其他的`Activity`。

## 2.整体流程
### 第一部分是从`Activity`到`AMS`的流程：
![从Activity到AMS](https://upload-images.jianshu.io/upload_images/3468445-d2a5dc9e9844abc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### startActivityForResult
[Activity.java]
`Activity`的`startActivity`最终会调用到`startActivityForResult`，其中调用了`Instrumentation`的`execStartActivity`方法
```
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {//没有父activity，一般都是这个情况
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            ......
        } else {//有父activity的情况，现在已经被fragment替代了
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```
#### execStartActivity
[Instrumentation.java]
```
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        ......
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            int result = ActivityManager.getService()//获取AMS代理对象，调用它的startActivity方法
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            //检查启动Activity的结果（抛出异常，例如清单文件未注册Activity）
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```
首先会调用`ActivityManager`的`getService`方法来获取`AMS`的代理对象，接着调用它的`startActivity`方法。这里与`Android 7.0`代码的逻辑有些不同，`Android 7.0`是通过`ActivityManagerNative`的`getDefault`来获取`AMS`的代理对象，现在这个逻辑封装到了`ActivityManager`中而不是`ActivityManagerNative`中。首先我们先来查看`ActivityManager`的`getService`方法做了什么：

#### getService
[ActivityManager.java]
```
public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                //得到IBinder类型的AMS的引用
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                //将它转换成IActivityManager类型的对象，这段代码采用的是AIDL
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```
也就是说`ActivityManager.getService().startActivity`方法实际跨进程调用到`AMS`的`startActivity`方法。

---------------------------------------------

### 第二部分
是从`AMS`到`ApplicationThread`，`ApplicationThread`运行在应用进程。在`startSpecificActivityLocked`方法中涉及到是否需要新建进程，但把需要新建进程的分支流程放到[进程创建流程](https://www.jianshu.com/writer#/notebooks/30354245/notes/35491911)中分析。
![从AMS到ApplicationThred](https://upload-images.jianshu.io/upload_images/3468445-bd4988be8deb9ed5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 这里开始的代码在`AMS`所在进程（`SystemServer`）处理，已经跨进程
#### startActivity
[ActivityManagerService.java]
```
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```
`retrun`了`startActivityAsUser`方法，`startActivityAsUser`方法比`startActivity`方法多了一个参数`UserHandle.getCallingUserId()`，这个方法会获得调用者的`UserId`，`AMS`会根据这个`UserId`来确定调用者的权限。

#### startActivityAsUser
[ActivityManagerService.java]
```
@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    //判断调用者进程是否被隔离  
    enforceNotIsolatedCaller("startActivity");
    //检查调用者权限
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
            userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    // 调用ActivityStarter.startActivityMayWait
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, "startActivityAsUser");
}
```
#### startActivityMayWait
[ActivityStarter.java]
```
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, String resolvedType,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, WaitResult outResult,
            Configuration globalConfig, Bundle bOptions, boolean ignoreTargetSecurity, int userId,
            TaskRecord inTask, String reason) {
        ...... 
        synchronized (mService) {
            ......
            int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                    aInfo, rInfo, voiceSession, voiceInteractor,
                    resultTo, resultWho, requestCode, callingPid,
                    callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                    options, ignoreTargetSecurity, componentSpecified, outRecord, inTask,
                    reason);
            ......
            return res;
        }
    }
```
`ActivityStarter`是`Android 7.0`新加入的类，它是加载`Activity`的控制类，会收集所有的逻辑来决定如何将`Intent`和`Flags`转换为`Activity`，并将`Activity`和`Task`以及`Stack`相关联。`ActivityStarter`的`startActivityMayWait`方法调用了`startActivityLocked`方法，如下所示。
#### startActivityLocked
[ActivityStarter.java]
```
int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, TaskRecord inTask, String reason) {

    if (TextUtils.isEmpty(reason)) {
        throw new IllegalArgumentException("Need to specify a reason.");
    }
    mLastStartReason = reason;
    mLastStartActivityTimeMs = System.currentTimeMillis();
    mLastStartActivityRecord[0] = null;

    mLastStartActivityResult = startActivity(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor, resultTo, resultWho, requestCode,
                callingPid, callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, mLastStartActivityRecord,
                inTask);

    if (outActivity != null) {
        outActivity[0] = mLastStartActivityRecord[0];
    }
    return mLastStartActivityResult != START_ABORTED ? mLastStartActivityResult : START_SUCCESS;
    }
```
#### startActivity
[ActivityStarter.java]
```
    private int startActivity(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
            String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
            String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
            ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
            ActivityRecord[] outActivity, TaskRecord inTask) {
        int err = ActivityManager.START_SUCCESS;
        final Bundle verificationBundle
                = options != null ? options.popAppVerificationBundle() : null;

        ProcessRecord callerApp = null;
        if (caller != null) {
            //获取调用者（它要启动Activity）的进程记录
            callerApp = mService.getRecordForAppLocked(caller);
            if (callerApp != null) {
                //获取调用者的pid和uid
                callingPid = callerApp.pid;
                callingUid = callerApp.info.uid;
            } else {
                Slog.w(TAG, "Unable to find app for caller " + caller
                        + " (pid=" + callingPid + ") when starting: "
                        + intent.toString());
                err = ActivityManager.START_PERMISSION_DENIED;
            }
        }
        ......
        //创建即将要启动的Activity的描述类ActivityRecord
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                mSupervisor, options, sourceRecord);
        ......
        doPendingActivityLaunchesLocked(false);

        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
                options, inTask, outActivity);
    }

```
#### startActivity
[ActivityStarter.java]
```
private int startActivity(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
        ActivityRecord[] outActivity) {
    int result = START_CANCELED;
    try {
        mService.mWindowManager.deferSurfaceLayout();
        result = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, outActivity);
    } finally {
       ......
    }

    postStartActivityProcessing(r, result, mSupervisor.getLastStack().mStackId,  mSourceRecord,
            mTargetStack);

    return result;
}
```
#### startActivityUnchecked
[ActivityStarter.java]
```
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {

        //ActivityStack的startActivityLocked,不要搞混了。
        //同时调用WindowManager准备App切换相关的工作
        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isFocusable()
                    || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                    && mStartActivity != topTaskActivity)) {
                ......
            } else {
                if (mTargetStack.isFocusable() && !mSupervisor.isFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityUnchecked");
                }
                //最终调用ActivityStackSupervisor的resumeFocusedStackTopActivityLocked
                mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                        mOptions);
            }
        } else {
            mTargetStack.addRecentActivityLocked(mStartActivity);
        }
        ......
        return START_SUCCESS;
    }
```
`startActivityUnchecked`方法主要处理栈管理相关的逻辑
#### resumeFocusedStackTopActivityLocked
[ActivityStackSupervisor.java]
```
boolean resumeFocusedStackTopActivityLocked(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!readyToResume()) {
            return false;
        }

        if (targetStack != null && isFocusedStack(targetStack)) {
            return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
        if (r == null || r.state != RESUMED) {
            mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
        } else if (r.state == RESUMED) {
            // Kick off any lingering app transitions form the MoveTaskToFront operation.
            mFocusedStack.executeAppTransition(targetOptions);
        }

        return false;
    }
```
#### resumeTopActivityUncheckedLocked
[ActivityStack.java]
```
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mStackSupervisor.inResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            mStackSupervisor.inResumeTopActivity = true;
            //调用了resumeTopActivityInnerLocked
            result = resumeTopActivityInnerLocked(prev, options);
        } finally {
            mStackSupervisor.inResumeTopActivity = false;
        }

        final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
        if (next == null || !next.canTurnScreenOn()) {
            checkReadyForSleep();
        }

        return result;
    }
```
#### resumeTopActivityInnerLocked
[ActivityStack.java]
```
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
    ......省略了很多代码
    } else {
        //关键是此行代码
        mStackSupervisor.startSpecificActivityLocked(next, true, true);
    }
    ......
    return true;
}
```
#### startSpecificActivityLocked
[ActivityStackSupervisor.java]
```
void startSpecificActivityLocked(ActivityRecord r,
        boolean andResume, boolean checkConfig) {
    // 获取即将要启动的Activity的所在的应用程序进程
    ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

    r.getStack().setLaunchTime(r);

    if (app != null && app.thread != null) {//已经运行的话
        try {
            if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                    || !"android".equals(r.info.packageName)) {
                app.addPackage(r.info.packageName, r.info.applicationInfo.versionCode,
                            mService.mProcessStats);
            }
            //如果应用程序已经运行，例如由本应用的MainActivity来启动此Activity
            realStartActivityLocked(r, app, andResume, checkConfig);
            return;
        } catch (RemoteException e) {
            ......
    }
    //如果进程不存在，则通过zygote创建应用进程
    mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
}
```
> `startSpecificActivityLocked`方法中涉及到是否需要创建进程，这里考虑的是如果进程已经存在的情况，把需要新建进程的分支流程放到[进程创建流程](https://www.jianshu.com/writer#/notebooks/30354245/notes/35491911)中分析。

#### realStartActivityLocked
[ActivityStackSupervisor.java]
```
final boolean realStartActivityLocked(ActivityRecord r, ProcessRecord app,
            boolean andResume, boolean checkConfig) throws RemoteException {
    ......
            app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    mergedConfiguration.getGlobalConfiguration(),
                    mergedConfiguration.getOverrideConfiguration(), r.compat,
                    r.launchedFromPackage, task.voiceInteractor, app.repProcState, r.icicle,
                    r.persistentState, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profilerInfo);
    ......
    }
```
这里的 `app.thread`指的是`IApplicationThread`，它的实现是`ActivityThread`的内部类`ApplicationThread`，其中`ApplicationThread`继承了`IApplicationThread.Stub`。`app`指的是传入的要启动的`Activity`所在的应用程序进程。

当前代码逻辑运行在`AMS`所在的进程（`SyetemServer`进程），通过`ApplicationThread`来与应用程序进程进行`Binder`通信，换句话说，`ApplicationThread`是`AMS`所在进程（`SyetemServer`进程）和应用程序进程的通信桥梁，如下图所示。 
![](https://upload-images.jianshu.io/upload_images/3468445-30a12a6e8df69d6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 第三部分
从`ApplicationThread`（`ActivityThread`内部类）开始的代码又回到了应用程序进程。`ApplicationThread`是`ActivityThread`的内部类，应用程序进程创建后会运行代表主线程的实例`ActivityThread`，它管理着当前应用程序进程的线程。
![](https://upload-images.jianshu.io/upload_images/3468445-4d5b503354fb0b75.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#### scheduleLaunchActivity
[ActivityThread.ApplicationThread.java]
```
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,
                ActivityInfo info, Configuration curConfig, Configuration overrideConfig,
                CompatibilityInfo compatInfo, String referrer, IVoiceInteractor voiceInteractor,
                int procState, Bundle state, PersistableBundle persistentState,
                List<ResultInfo> pendingResults, List<ReferrerIntent> pendingNewIntents,
                boolean notResumed, boolean isForward, ProfilerInfo profilerInfo) {

            updateProcessState(procState, false);
            ActivityClientRecord r = new ActivityClientRecord();
            r.token = token;
            r.ident = ident;
            r.intent = intent;
            r.referrer = referrer;
            r.voiceInteractor = voiceInteractor;
            r.activityInfo = info;
            r.compatInfo = compatInfo;
            r.state = state;
            ......
            updatePendingConfiguration(curConfig);
            sendMessage(H.LAUNCH_ACTIVITY, r);
        }
```
`scheduleLaunchActivity`方法会将启动`Activity`的参数封装成`ActivityClientRecord` ，`sendMessage`方法向`H`类发送类型为`LAUNCH_ACTIVITY`的消息，并将`ActivityClientRecord` 传递过去，`sendMessage`方法有多个重载方法，最终调用的`sendMessage`方法如下所示。
#### sendMessage
[ActivityThread.java]
```
private void sendMessage(int what, Object obj, int arg1, int arg2, boolean async) {
        if (DEBUG_MESSAGES) Slog.v(
            TAG, "SCHEDULE " + what + " " + mH.codeToString(what)
            + ": " + arg1 + " / " + obj);
        Message msg = Message.obtain();
        msg.what = what;
        msg.obj = obj;
        msg.arg1 = arg1;
        msg.arg2 = arg2;
        if (async) {
            msg.setAsynchronous(true);
        }
        mH.sendMessage(msg);
    }
```
这里的`H`类是`ActivityThread`内部类并继承`Handler`，是应用程序进程中主线程的消息管理类。`mH.sendMessage(msg)`发送的消息最终会在`H`类的`handleMessage`方法中处理
#### H类handleMessage方法
```
private class H extends Handler {
      public static final int LAUNCH_ACTIVITY         = 100;
      public static final int PAUSE_ACTIVITY          = 101;
...
public void handleMessage(Message msg) {
          if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
          switch (msg.what) {
              case LAUNCH_ACTIVITY: {
                  Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                  final ActivityClientRecord r = (ActivityClientRecord) msg.obj;//1
                  r.packageInfo = getPackageInfoNoCheck(
                          r.activityInfo.applicationInfo, r.compatInfo);//2
                  handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");//3
                  Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
              } break;
              case RELAUNCH_ACTIVITY: {
                  Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                  ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                  handleRelaunchActivity(r);
                  Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
              } break;
            ...
}
```
其中对`LAUNCH_ACTIVITY`消息的处理，注释1处先获得了`msg`中的`obj`。
在注释2处通过`getPackageInfoNoCheck`方法获得`LoadedApk`类型的对象并赋值给`ActivityClientRecord` 的成员变量`packageInfo` 。应用程序进程要启动`Activity`时需要将该`Activity`所属的`APK`加载进来，而`LoadedApk`就是用来描述已加载的`APK`文件。 
在注释3处调用`handleLaunchActivity`方法
#### handleLaunchActivity
[ActivityThread.java]
```
    private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent, String reason) {
        ......
        WindowManagerGlobal.initialize();
        //启动Activity
        Activity a = performLaunchActivity(r, customIntent);
        if (a != null) {
            r.createdConfig = new Configuration(mConfiguration);
            reportSizeConfigurations(r);
            Bundle oldState = r.state;
            //内部会调用Activity的onResume()方法
            handleResumeActivity(r.token, false, r.isForward,
                    !r.activity.mFinished && !r.startsNotResumed, r.lastProcessedSeq, reason);
            if (!r.activity.mFinished && r.startsNotResumed) {
                performPauseActivityIfNeeded(r, reason);
                if (r.isPreHoneycomb()) {
                    r.state = oldState;
                }
            }
        } else {
            // 如果有错误，停止启动
            try {
                ActivityManager.getService()
                    .finishActivity(r.token, Activity.RESULT_CANCELED, null,
                            Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
        }
    }
```
> **onResume()和绘制视图相关**：其中`handleResumeActivity`方法内部的`performResumeActivity`会间接调用`Activity`的`onResume()`方法。且`handleResumeActivity`内部在`performResumeActivity`执行后（也是`onResume`后），会调用`ViewManager`（--->WindowManager--->WindowManagerImpl）的`addView`方法，进而调用到`WindowManagerGlobal`的`addView`，其中会调用`ViewRootImpl`的`setView`方法--->`requestLayout()`--->`scheduleTraversals()`。
所以在`onResume()`后才会开始绘制视图。
（在`Window`章节中有提到）

#### performLaunchActivity
[ActivityThread.java]
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        //1.获取ActivityInfo类
        ActivityInfo aInfo = r.activityInfo;//1
        if (r.packageInfo == null) {
        //2。获取APK文件的描述类LoadedApk
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        //3.组件名称:Activity全类名
        ComponentName component = r.intent.getComponent();//3
        ......
        //4.创建要启动Activity的上下文环境
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            //5.用类加载器通过反射创建该Activity的实例
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            ......
        } catch (Exception e) {
            ......
        }

        try {
            //6.创建Application，内部会调用Application的onCreate方法
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);//6
            ......
            if (activity != null) {
                ......
                //7.调用Activity的attach()方法，初始化Activity，会创建
                //Window对象（PhoneWindow）并与Activity自身进行关联
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                ......
                if (theme != 0) {
                    activity.setTheme(theme); //设置Activity主题
                }
                activity.mCalled = false;
                //8.启动Activity，其中会调用Activity的onCreate()方法
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                ......
                r.activity = activity;
                r.stopped = true;
                if (!r.activity.mFinished) {
                    //9.内部会调用Activity的onStart()方法
                    activity.performStart();
                    r.stopped = false;
                }
                    ......
                }
            }
            r.paused = true;
            mActivities.put(r.token, r);
        }......
        return activity;
    }
```
1. 获取`ActivityInfo`类，`ActivityInfo`用于存储代码和`AndroidManifes`设置的`Activity`和`receiver`节点信息，比如`Activity`的`theme`和`launchMode`。
2. 获取`APK`文件的描述类`LoadedApk`
3. 获取要启动的`Activity`的`ComponentName`类，`ComponentName`类中保存了该`Activity`的包名和类名
4. 创建要启动`Activity`的上下文环境
5. 根据`ComponentName`中存储的`Activity`类名，用类加载器通过反射创建该`Activity`的实例
6. 创建`Application`，内部会调用`Application`的`onCreate`方法
7. 调用`Activity`的`attach()`方法，初始化`Activity`，会创建`Window`对象（`PhoneWindow`）并与`Activity`自身进行关联
8. 启动`Activity`，其中会调用`Activity`的`onCreate()`方法
9. `activity.performStart()`内部会调用`Activity`的`onStart()`方法

本方法返回后，会在上层`handleLaunchActivity`方法中调用的`handleResumeActivity`方法中调用`Activity`的`onResume()`方法

#### callActivityOnCreate
[Instrumentation.java]
```
public void callActivityOnCreate(Activity activity, Bundle icicle) {
    prePerformCreate(activity);
    activity.performCreate(icicle);//内部调用performCreate(icicle, null)
    postPerformCreate(activity);
}
```
#### performCreate
[Activity.java]
```
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        mCanEnterPictureInPicture = true;
        restoreHasCurrentPermissionRequest(icicle);
        //这里调用了Activity的onCreate方法
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        mActivityTransitionState.readState(icicle);

        mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mFragments.dispatchActivityCreated();
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    }
```
在`Activity`的`performCreate`方法中调用了`onCreate`，启动过程分析结束。启动后就是在`onCreate`方法中的`setContent`方法来设置界面。

而关于`onStart`和`onResume`在上面均有提及，但就不要再列出方法代码了。

#### 关于引发视图绘制的总结
`onCreate()`中调用`setContentView()`会创建好`DecorView`对象和自己写的布局对应的`View`对象树。

在`onResume()`执行后，会调用`ViewManager`（--->`WindowManager`--->`WindowManagerImpl`）的`addView`方法，进而调用到`WindowManagerGlobal`的`addView`，其中会调用`ViewRootImpl`的`setView`方法--->`requestLayout()`--->`scheduleTraversals()`，从而从`DecorView`开始引发绘制。
