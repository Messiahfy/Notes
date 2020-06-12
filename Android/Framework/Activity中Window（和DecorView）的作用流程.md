## 1.概述
`Activity`对象都有一个`Window`成员变量，它在`Activity`的创建过程中的作用分析

* `Activity`的`attach`阶段会创建`PhoneWindow`，并给`PhoneWindow`设置`WindowMnager`（`WindowManagerImpl`），而每个`WindowMnager`内部都可以访问到`WindowManagerGlobal`

* `Activity`的`onCreate`阶段的`setContentView`会创建`DecorView`，并与`PhoneWindow`关联（并未真正加入）

* `Activity`的`onResume`阶段之后，会调用`windowManagerIpml`的`addView`，实际调用的是`Global的addView`方法。会为`DecorView`创建一个对应的`ViewRootImpl`。并调用`ViewRootImpl`的`setView`传入`DecorView`，则引发从`DecorView`为根开始的视图树绘制

![大概结构图](https://upload-images.jianshu.io/upload_images/3468445-349c0536dc9552ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 如果主动调用`WindowManager.addView`，则会再创建一个`ViewRootImpl`，因为现在`PhoneWindow`下不止`DecorView`一个视图树，还有一个`addView`中传入的`view`为根的视图树。

## 2.流程
### 2.1 Window和DecorView的创建
在`Activity`启动过程的分析中，第二部分的`ActivityThread`类的`performLaunchActivity`方法中调用了`Activity`的`attach`方法，此方法中会为`Activity`创建`Window`（`PhoneWindow`）对象。
#### attach
[Activity.java]
```
final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        //绑定Context，由上级方法创建传来
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);
        //创建了PhoneWindow，mWindow为Activity的成员变量
        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);//对Window设置callback为Activity，必要时候可以回调Activity
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }
        //为Window设置WindowManager
        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
    }
```
#### `Window`和`Activity`、`View`的各种作用关系流程如下：
1. `Activity`的`attach`方法创建了`Window`（`PhoneWindow`）。

2. 调用`attach`方法后，会调用`ActivityThread`的`callActivityOnCreate`方法直到`Activity`的`onCreate`方法。在`Activity`的`onCreate`方法里，我们调用`setContentView()`会调用到`PhoneWindow`的`setContentView()`方法，其中会调用`installDecor()`构造`DecorView`。

>（假设一个`Activity`不调用`setContentView`，那么开始不会调用`installDecor()`来创建`DecorView`，但是在绘制前会调用`PhoneWindow`的`getDecorView()`方法，方法内判断`DecorView`为`null`则会调用`installDecor()`方法创建。所以不管`Activity`是否调用`setContentView()`，都会创建`DecorView`，只是`DecorView`的`id`为`content`的`ViewGroup`没有子布局）
```
public void setContentView(int layoutResID) {
        if (mContentParent == null) {
            installDecor();//构造设置DecorView
        } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            mContentParent.removeAllViews();
        }

        if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
            final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                    getContext());
            transitionTo(newScene);
        } else {
            //构造设置完DecorView后，mContentParent就是DecorView中的android.R.id.content
            //把我们在setContent传入的view放到mContentParent中
            mLayoutInflater.inflate(layoutResID, mContentParent);
        }
        mContentParent.requestApplyInsets();
        final Callback cb = getCallback();
        if (cb != null && !isDestroyed()) {
            cb.onContentChanged();//内容布局设置后回调Activity
        }
        mContentParentExplicitlySet = true;
    }
```
将`view`添加到`DecorView`的`mContentParent`中，也就是将资源布局文件和`phoneWindow`关联。

#### installDecor
[PhoneWindow.java]
上面说`installDecor()`方法构造和设置DecorView，这里看此方法的源码
`DecorView`继承`FrameLayout`
```
private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            mDecor = generateDecor(-1);//返回DecorView的实例
            mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            mDecor.setIsRootNamespace(true);
            if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
                mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
            }
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            //此方法会把Android自带的资源布局作为DecorView的子布局，并把其中的 
            //android.R.id.content返回赋值给mContentParent
            mContentParent = generateLayout(mDecor);
            ......
        }
        ......
    }
```
（以有`titleBar`的情况为例：`DecorView`的子布局是`Android SDK中`的`res\layout\screen_title.xml`，具体情况查看省略了的代码）

从上述过程可知，`Window`和`DecorView`已经创建完成，`Activity`的布局文件也添加到了`DecorView`中的`R.id.content`（`mContentParent`），但`DecorView`还没有正式添加到`Window`中，视图也没有开始绘制。
结构如下：
![](https://upload-images.jianshu.io/upload_images/3468445-75579259908a07b6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

--------------------------------------------
### 2.2 后续流程将引发绘制

#### handleResumeActivity
[ActivityThread.java]
`ActivityThread`类的`handleLaunchActivity`方法内部在执行完`performLaunchActivity`方法后，要执行`handleResumeActivity `方法，此方法内部会执行`performResumeActivity`进而执行到`Activity`的`onResume`。`handleResumeActivity`方法在`performResumeActivity`执行完后（也是`onResume`执行完），会执行下面的代码：
```
if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (r.mPreserveWindow) {
                    a.mWindowAdded = true;
                    r.mPreserveWindow = false;
                    ViewRootImpl impl = decor.getViewRootImpl();
                    if (impl != null) {
                        impl.notifyChildRebuilt();
                    }
                }
                if (a.mVisibleFromClient) {
                    if (!a.mWindowAdded) {
                        a.mWindowAdded = true;
                        wm.addView(decor, l);//把decorView添加到Window中，关键在此
                    } else {
                        a.onWindowAttributesChanged(l);
                    }
                }
            }
```
`Activity`的`attach`方法中设置了`WindowManager`，实际是其实现类`WindowManagerImpl`，所以这里就是调用了`WindowManagerImpl`的`addView`方法，把`DecorView`添加到`Window`中。

#### addView
[WindowManagerImpl.java]
`WindowManagerImpl`的`addView`方法如下，调用了`WindowManagerGlobal`的`addView`
```
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    applyDefaultToken(params);
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```
#### addView
[WindowManagerGlobal.java]
**每个DecorView对应一个ViewRootImpl**
```
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        ......
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        ......
        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            ......
            //创建ViewRootImpl，并且将view（DecorView）与之绑定
            root = new ViewRootImpl(view.getContext(), display);
            view.setLayoutParams(wparams);//设置布局参数
            mViews.add(view);//将当前view添加到mViews集合中
            mRoots.add(root);//将当前ViewRootImpl添加到mRoots集合中
            mParams.add(wparams);//将当前window的params添加到mParams集合中

            // 最后调用ViewRootImpl的setView方法
            try {
                root.setView(view, wparams, panelParentView);
            } ......
        }
    }
```
其中关于`WindowManagerGlobal`有如下几个重要的集合：
```
//存储所有Window对应的View
private final ArrayList<View> mViews = new ArrayList<View>();

 //存储所有Window对应的ViewRootImpl
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();

//存储所有Window对应的布局参数
private final ArrayList<WindowManager.LayoutParams> mParams = new ArrayList<WindowManager.LayoutParams>(); 

//存储正被删除的View对象（已经调用removeView但是还未完成删除操作的Window对象）     
private final ArraySet<View> mDyingViews = new ArraySet<View>();
```

#### setView
[ViewRootImpl.java]
在`ViewRootImpl`的`setView`方法中，将引发完成`View`的绘制和开始接受触摸事件
```
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                mView = view;
                ......
                requestLayout();//引发View的绘制流程
                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    //创建InputChannel
                    mInputChannel = new InputChannel();
                }
                try {
                    //通过WindowSession进行IPC调用，将View添加到Window上
                    //mWindow即W类，用来接收WmS信息
                    //同时通过InputChannel接收触摸事件回调
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
                }
                ......
                    //处理触摸事件回调
                    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
                ......
                }
            }
        }
```
`requestLayout()`引发View的绘制流程，具体在Android View中分析。

