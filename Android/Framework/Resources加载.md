Android 4.4源码

## 1. 组件持有Resources的流程
### Application的Context持有Resources的过程

ActivityThread#handleBindApplication
```
private void handleBindApplication(AppBindData data) {
    ...
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    ...
}
```

LoadApk#makeApplication
```
public Application makeApplication(boolean forceDefaultAppClass,
    Instrumentation instrumentation) {
    if (mApplication != null) {
        return mApplication;
    }

    Application app = null;

    String appClass = mApplicationInfo.className;
    if (forceDefaultAppClass || (appClass == null)) {
        appClass = "android.app.Application";
    }

    try {
        java.lang.ClassLoader cl = getClassLoader();
        // 关键在于这一步创建Context
        ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
        app = mActivityThread.mInstrumentation.newApplication(
                cl, appClass, appContext); // Context会传给Application
        appContext.setOuterContext(app);
    } catch (Exception e) {
        ...
    }
    mActivityThread.mAllApplications.add(app);
    mApplication = app;

    if (instrumentation != null) {
        try {
            instrumentation.callApplicationOnCreate(app);
        } catch (Exception e) {
            ...
        }
    }    
    return app;
}
```
LoadApk表示当前apk，一个Android应用程序可以有多个apk，可能是因为这个原因，所以针对apk也有对应的数据结构。

ContextImpl#createAppContext
```
static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
    return new ContextImpl(null, mainThread,
            packageInfo, null, null, false, null, null);
}

private ContextImpl(ContextImpl container, ActivityThread mainThread,
        LoadedApk packageInfo, IBinder activityToken, UserHandle user, boolean restricted,
        Display display, Configuration overrideConfiguration) {
    mOuterContext = this;

    mMainThread = mainThread;
    mActivityToken = activityToken;
    mRestricted = restricted;

    if (user == null) {
        user = Process.myUserHandle();
    }
    mUser = user;

    mPackageInfo = packageInfo;
    mContentResolver = new ApplicationContentResolver(this, mainThread, user);
    mResourcesManager = ResourcesManager.getInstance();
    mDisplay = display;
    mOverrideConfiguration = overrideConfiguration;

    final int displayId = getDisplayId();
    CompatibilityInfo compatInfo = null;
    if (container != null) {
        compatInfo = container.getDisplayAdjustments(displayId).getCompatibilityInfo();
    }
    if (compatInfo == null && displayId == Display.DEFAULT_DISPLAY) {
        compatInfo = packageInfo.getCompatibilityInfo();
    }
    mDisplayAdjustments.setCompatibilityInfo(compatInfo);
    mDisplayAdjustments.setActivityToken(activityToken);

    // 创建Resoures
    Resources resources = packageInfo.getResources(mainThread);
    if (resources != null) {
        // Activity才会执行这里
        if (activityToken != null
                || displayId != Display.DEFAULT_DISPLAY
                || overrideConfiguration != null
                || (compatInfo != null && compatInfo.applicationScale
                        != resources.getCompatibilityInfo().applicationScale)) {
            resources = mResourcesManager.getTopLevelResources(
                    packageInfo.getResDir(), displayId,
                    overrideConfiguration, compatInfo, activityToken);
        }
    }
    mResources = resources;
    ...
}
```

LoadApk#getResources
```
public Resources getResources(ActivityThread mainThread) {
    if (mResources == null) {
        mResources = mainThread.getTopLevelResources(mResDir,
                Display.DEFAULT_DISPLAY, null, this);
    }
    return mResources;
}
```

ActivityThread#getTopLevelResources
```
Resources getTopLevelResources(String resDir,
        int displayId, Configuration overrideConfiguration,
        LoadedApk pkgInfo) {
    return mResourcesManager.getTopLevelResources(resDir, displayId, overrideConfiguration,
            pkgInfo.getCompatibilityInfo(), null);
}
```

ResourcesManager#getTopLevelResources
```
public Resources getTopLevelResources(String resDir, int displayId,
        Configuration overrideConfiguration, CompatibilityInfo compatInfo, IBinder token) {
    final float scale = compatInfo.applicationScale;
    ResourcesKey key = new ResourcesKey(resDir, displayId, overrideConfiguration, scale,
            token);
    Resources r;
    synchronized (this) {
        // 取缓存
        WeakReference<Resources> wr = mActiveResources.get(key);
        r = wr != null ? wr.get() : null;
        if (r != null && r.getAssets().isUpToDate()) {
            return r;
        }
    }

    AssetManager assets = new AssetManager();
    // 添加资源目录到AssetManager
    if (assets.addAssetPath(resDir) == 0) {
        return null;
    }

    DisplayMetrics dm = getDisplayMetricsLocked(displayId);
    Configuration config;
    boolean isDefaultDisplay = (displayId == Display.DEFAULT_DISPLAY);
    final boolean hasOverrideConfig = key.hasOverrideConfiguration();
    if (!isDefaultDisplay || hasOverrideConfig) {
        config = new Configuration(getConfiguration());
        if (!isDefaultDisplay) {
            applyNonDefaultDisplayMetricsToConfigurationLocked(dm, config);
        }
        if (hasOverrideConfig) {
            config.updateFrom(key.mOverrideConfiguration);
        }
    } else {
        config = getConfiguration();
    }
    // 没有缓存就重新创建
    r = new Resources(assets, dm, config, compatInfo, token);

    synchronized (this) {
        ...
        mActiveResources.put(key, new WeakReference<Resources>(r));
        return r;
    }
}
```

到这里，Application的Context已经持有Resources

## Activity的Context持有Resources的过程

ActivityThread#performLaunchActivity
```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
    ...
    ContextImpl appContext = createBaseContextForActivity(r); //这个context会传给Activity
    ...
}
```

ActivityThread#createBaseContextForActivity
```
private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
    ...
    ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
    ...
    return appContext;
}
```
ContextImpl#createActivityContext
```
static ContextImpl createActivityContext(ActivityThread mainThread,
        LoadedApk packageInfo, IBinder activityToken) {
    ...
    return new ContextImpl(null, mainThread,
            packageInfo, activityToken, null, false, null, null);
}
```
同样是执行ContextImpl的构造函数

## Service的Context持有Resources的过程

ActivityThread#handleCreateService
```
private void handleCreateService(CreateServiceData data) {
    ...
    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
    ...
}
```
跟Application是相同的流程

> BroadcastReceiver的onReceive函数的context是Application的context

## 使用Resources获取资源
1. getString-->getText
```
public CharSequence getText(int id, CharSequence def) {
    CharSequence res = id != 0 ? mAssets.getResourceText(id) : null;
    return res != null ? res : def;
}
```

AssetManager#getResourceText
```
final CharSequence getResourceText(int ident) {
    synchronized (this) {
        TypedValue tmpValue = mValue;
        int block = loadResourceValue(ident, (short) 0, tmpValue, true); //native方法
        if (block >= 0) {
            if (tmpValue.type == TypedValue.TYPE_STRING) {
                return mStringBlocks[block].get(tmpValue.data);
            }
            return tmpValue.coerceToString();
        }
    }
    return null;
}
```
通过native方法，把数据存到tmpValue中，然后返回

2. 获取Drawable
```
public Drawable getDrawable(int id) throws NotFoundException {
    TypedValue value;
    synchronized (mAccessLock) {
        value = mTmpValue;
        if (value == null) {
            value = new TypedValue();
        } else {
            mTmpValue = null;
        }
        getValue(id, value, true); //也会调用到native方法，并把数据存到TypedValue
    }
    Drawable res = loadDrawable(value, id); // 通过value的数据和id，创建Drawable
    synchronized (mAccessLock) {
        if (mTmpValue == null) {
            mTmpValue = value;
        }
    }
    return res;
}
```