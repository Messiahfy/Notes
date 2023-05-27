SurfaceView主要在游戏、相机、视频这类场景中使用，这类场景的特点是绘制的内容会被动的频繁刷新，而普通的View一般只在用户触摸操作的时候刷新。

一个窗口（Activity、Dialog、PopupWindow）内所有的普通View共享一个Surface，该Surface保存在ViewRootImpl内，而SurfaceView会关联一个属于自己的Surface。DecorView，也就是根结点视图，它在WMS中有一个对应的WindowState，在SurfaceFlinger中对应一个Layer；SurfaceView自带一个Surface，这个Surface在WMS中有自己对应的WindowState，在SF中也会有自己的Layer。


在同一个Window下，有一个作为播放器的SurfaceView A，显示层级上，它在View B的上面，还有一个封面View C盖在SurfaceView上面，ViewB和封面C都归属于Activity的Window对应的Surface，而作为播放器的SurfaceView A则是另一个Surface，此时SurfaceFlinger应该怎合成这两个Surface？你会发现这俩Surface无论谁上谁下都不对，因为SurfaceView不是单纯地在Window的上面或下面，而是在Window里面的一个层里。

为解决上述问题，Android特地为SurfaceView设计了一套"挖洞"机制。首先在SurfaceFlinger中，SurfaceView对应的Surface的Layer层级是比其宿主Window对应的Surface的Layer层级要低的，也就是说宿主Window的Surface是盖在SurfaceView的Surface上面的，在SurfaceView执行onAttachToWindow()时，会调用父View的requestTransparentRegion()，该函数会指定哪块区域需要被设为透明，当ViewRoot执行performTraversals()进行测量定位绘制时，能拿到这块区域并将其设为透明。当宿主Window的Surface对应区域透明后，盖在其下方的SurfaceView的Surface自然就能够被看到了，设置透明区域的操作被形象地称之为"挖孔"。


View和SurfaceView的区别:

1. View适用于主动更新的情况，而SurfaceView则适用于被动更新的情况，比如频繁刷新界面。
2. View在主线程中对页面进行刷新，而SurfaceView则开启一个子线程来对页面进行刷新。
3. View在绘图时没有实现双缓冲机制，SurfaceView在底层机制中就实现了双缓冲机制。

模板代码：
```
public class SurfaceViewTemplate extends SurfaceView implements SurfaceHolder.Callback, Runnable {
    private SurfaceHolder mSurfaceHolder;
    //绘图的Canvas
    private Canvas mCanvas;
    //子线程标志位
    private boolean mIsDrawing;
    public SurfaceViewTemplate(Context context) {
        this(context, null);
    }

    public SurfaceViewTemplate(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public SurfaceViewTemplate(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        initView();
    }

    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        mIsDrawing = true;
        //开启子线程
        new Thread(this).start();
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {

    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        mIsDrawing = false;
    }

    @Override
    public void run() {
        while (mIsDrawing){
            drawSomething();
        }
    }
    //绘图逻辑
    private void drawSomething() {
        try {
            //获得canvas对象
            mCanvas = mSurfaceHolder.lockCanvas();
            //绘制背景
            mCanvas.drawColor(Color.WHITE);
            //绘图
        }catch (Exception e){

        }finally {
            if (mCanvas != null){
                //释放canvas对象并提交画布
                mSurfaceHolder.unlockCanvasAndPost(mCanvas);
            }
        }
    }

    /**
     * 初始化View
     */
    private void initView(){
        mSurfaceHolder = getHolder();
        mSurfaceHolder.addCallback(this);
        setFocusable(true);
        setKeepScreenOn(true);
        setFocusableInTouchMode(true);
    }
}
```
* Activity的onResume()方法之后才会执行Surface的创建方法surfaceCreated()，当Activity执行onPause()方法后会执行Surface的surfaceDestroyed()方法
* surfaceView的onDraw()方法不会被调用，因为setWillNotDraw(true)

## Android 9 版本源码分析
```
    protected void updateSurface() {
        if (!mHaveFrame) {
            return;
        }
        ViewRootImpl viewRoot = getViewRootImpl();
        if (viewRoot == null || viewRoot.mSurface == null || !viewRoot.mSurface.isValid()) {
            return;
        }

        mTranslator = viewRoot.mTranslator;
        if (mTranslator != null) {
            mSurface.setCompatibilityTranslator(mTranslator);
        }

        int myWidth = mRequestedWidth;
        if (myWidth <= 0) myWidth = getWidth();
        int myHeight = mRequestedHeight;
        if (myHeight <= 0) myHeight = getHeight();

        final boolean formatChanged = mFormat != mRequestedFormat;
        final boolean visibleChanged = mVisible != mRequestedVisible;
        final boolean creating = (mSurfaceControl == null || formatChanged || visibleChanged)
                && mRequestedVisible;
        final boolean sizeChanged = mSurfaceWidth != myWidth || mSurfaceHeight != myHeight;
        final boolean windowVisibleChanged = mWindowVisibility != mLastWindowVisibility;
        boolean redrawNeeded = false;

        // 这几种情况都需要刷新
        if (creating || formatChanged || sizeChanged || visibleChanged || windowVisibleChanged) {
            getLocationInWindow(mLocation);

            try {
                final boolean visible = mVisible = mRequestedVisible;
                mWindowSpaceLeft = mLocation[0];
                mWindowSpaceTop = mLocation[1];
                mSurfaceWidth = myWidth;
                mSurfaceHeight = myHeight;
                mFormat = mRequestedFormat;
                mLastWindowVisibility = mWindowVisibility;

                mScreenRect.left = mWindowSpaceLeft;
                mScreenRect.top = mWindowSpaceTop;
                mScreenRect.right = mWindowSpaceLeft + getWidth();
                mScreenRect.bottom = mWindowSpaceTop + getHeight();
                if (mTranslator != null) {
                    mTranslator.translateRectInAppWindowToScreen(mScreenRect);
                }

                final Rect surfaceInsets = getParentSurfaceInsets();
                mScreenRect.offset(surfaceInsets.left, surfaceInsets.top);

                if (creating) {
                    // 创建SurfaceSession，作为和surface flinger的连接（这里传入viewRoot的surface用于表示是surfaceView的surface的parent，具体作用暂不深究）
                    mSurfaceSession = new SurfaceSession(viewRoot.mSurface);
                    // 当前的SurfaceControl将被销毁，也就是会更新mSurfaceControl
                    mDeferredDestroySurfaceControl = mSurfaceControl;

                    updateOpaqueFlag();
                    final String name = "SurfaceView - " + viewRoot.getTitle().toString();

                    // 更新mSurfaceControl，SurfaceControlWithBackground是SurfaceControl的代理
                    mSurfaceControl = new SurfaceControlWithBackground(
                            name,
                            (mSurfaceFlags & SurfaceControl.OPAQUE) != 0,
                            // 实际就是这里的Builder会创建实际的SurfaceControl，SurfaceControl持有SurfaceSession
                            new SurfaceControl.Builder(mSurfaceSession)
                                    .setSize(mSurfaceWidth, mSurfaceHeight)
                                    .setFormat(mFormat)
                                    .setFlags(mSurfaceFlags));
                } else if (mSurfaceControl == null) {
                    return;
                }

                boolean realSizeChanged = false;

                mSurfaceLock.lock(); // 锁
                try {
                    mDrawingStopped = !visible;

                    if (DEBUG) Log.i(TAG, System.identityHashCode(this) + " "
                            + "Cur surface: " + mSurface);

                    SurfaceControl.openTransaction();
                    try {
                        mSurfaceControl.setLayer(mSubLayer);
                        if (mViewVisibility) {
                            mSurfaceControl.show();
                        } else {
                            mSurfaceControl.hide();
                        }

                        // While creating the surface, we will set it's initial
                        // geometry. Outside of that though, we should generally
                        // leave it to the RenderThread.
                        //
                        // There is one more case when the buffer size changes we aren't yet
                        // prepared to sync (as even following the transaction applying
                        // we still need to latch a buffer).
                        // b/28866173
                        if (sizeChanged || creating || !mRtHandlingPositionUpdates) {
                            mSurfaceControl.setPosition(mScreenRect.left, mScreenRect.top);
                            mSurfaceControl.setMatrix(mScreenRect.width() / (float) mSurfaceWidth,
                                    0.0f, 0.0f,
                                    mScreenRect.height() / (float) mSurfaceHeight);
                        }
                        if (sizeChanged) {
                            mSurfaceControl.setSize(mSurfaceWidth, mSurfaceHeight);
                        }
                    } finally {
                        SurfaceControl.closeTransaction();
                    }

                    if (sizeChanged || creating) {
                        redrawNeeded = true;
                    }

                    mSurfaceFrame.left = 0;
                    mSurfaceFrame.top = 0;
                    if (mTranslator == null) {
                        mSurfaceFrame.right = mSurfaceWidth;
                        mSurfaceFrame.bottom = mSurfaceHeight;
                    } else {
                        float appInvertedScale = mTranslator.applicationInvertedScale;
                        mSurfaceFrame.right = (int) (mSurfaceWidth * appInvertedScale + 0.5f);
                        mSurfaceFrame.bottom = (int) (mSurfaceHeight * appInvertedScale + 0.5f);
                    }

                    final int surfaceWidth = mSurfaceFrame.right;
                    final int surfaceHeight = mSurfaceFrame.bottom;
                    realSizeChanged = mLastSurfaceWidth != surfaceWidth
                            || mLastSurfaceHeight != surfaceHeight;
                    mLastSurfaceWidth = surfaceWidth;
                    mLastSurfaceHeight = surfaceHeight;
                } finally {
                    mSurfaceLock.unlock();
                }

                try {
                    redrawNeeded |= visible && !mDrawFinished;

                    SurfaceHolder.Callback callbacks[] = null;

                    final boolean surfaceChanged = creating;

                    // 此前已创建了surface，现在重新创建或者变得不可见，则要先销毁surface
                    if (mSurfaceCreated && (surfaceChanged || (!visible && visibleChanged))) {
                        mSurfaceCreated = false;
                        if (mSurface.isValid()) {
                            callbacks = getSurfaceCallbacks();
                            for (SurfaceHolder.Callback c : callbacks) {
                                // 回调surfaceDestroyed
                                c.surfaceDestroyed(mSurfaceHolder);
                            }
                            // Since Android N the same surface may be reused and given to us
                            // again by the system server at a later point. However
                            // as we didn't do this in previous releases, clients weren't
                            // necessarily required to clean up properly in
                            // surfaceDestroyed. This leads to problems for example when
                            // clients don't destroy their EGL context, and try
                            // and create a new one on the same surface following reuse.
                            // Since there is no valid use of the surface in-between
                            // surfaceDestroyed and surfaceCreated, we force a disconnect,
                            // so the next connect will always work if we end up reusing
                            // the surface.
                            if (mSurface.isValid()) {
                                mSurface.forceScopedDisconnect();
                            }
                        }
                    }
                    
                    // 从SurfaceContro复制Surface到当前的，持有对原始surface相同数据的引用
                    if (creating) {
                        mSurface.copyFrom(mSurfaceControl);
                    }

                    if (sizeChanged && getContext().getApplicationInfo().targetSdkVersion
                            < Build.VERSION_CODES.O) {
                        // Some legacy applications use the underlying native {@link Surface} object
                        // as a key to whether anything has changed. In these cases, updates to the
                        // existing {@link Surface} will be ignored when the size changes.
                        // Therefore, we must explicitly recreate the {@link Surface} in these
                        // cases.
                        mSurface.createFrom(mSurfaceControl);
                    }

                    if (visible && mSurface.isValid()) {
                        // 回调 surfaceCreated
                        if (!mSurfaceCreated && (surfaceChanged || visibleChanged)) {
                            mSurfaceCreated = true;
                            mIsCreating = true;
                            if (DEBUG) Log.i(TAG, System.identityHashCode(this) + " "
                                    + "visibleChanged -- surfaceCreated");
                            if (callbacks == null) {
                                callbacks = getSurfaceCallbacks();
                            }
                            for (SurfaceHolder.Callback c : callbacks) {
                                c.surfaceCreated(mSurfaceHolder);
                            }
                        }
                        // 回调 surfaceChanged
                        if (creating || formatChanged || sizeChanged
                                || visibleChanged || realSizeChanged) {
                            if (callbacks == null) {
                                callbacks = getSurfaceCallbacks();
                            }
                            for (SurfaceHolder.Callback c : callbacks) {
                                c.surfaceChanged(mSurfaceHolder, mFormat, myWidth, myHeight);
                            }
                        }
                        if (redrawNeeded) {
                            if (callbacks == null) {
                                callbacks = getSurfaceCallbacks();
                            }

                            mPendingReportDraws++;
                            viewRoot.drawPending();
                            SurfaceCallbackHelper sch =
                                    new SurfaceCallbackHelper(this::onDrawFinished);
                            sch.dispatchSurfaceRedrawNeededAsync(mSurfaceHolder, callbacks);
                        }
                    }
                } finally {
                    mIsCreating = false;
                    if (mSurfaceControl != null && !mSurfaceCreated) {
                        mSurface.release();

                        mSurfaceControl.destroy();
                        mSurfaceControl = null;
                    }
                }
            } catch (Exception ex) {
                //...
            }
        } else {
            // 其他变化情况
            // Calculate the window position in case RT loses the window
            // and we need to fallback to a UI-thread driven position update
            getLocationInSurface(mLocation);
            final boolean positionChanged = mWindowSpaceLeft != mLocation[0]
                    || mWindowSpaceTop != mLocation[1];
            final boolean layoutSizeChanged = getWidth() != mScreenRect.width()
                    || getHeight() != mScreenRect.height();
            if (positionChanged || layoutSizeChanged) { // Only the position has changed
                mWindowSpaceLeft = mLocation[0];
                mWindowSpaceTop = mLocation[1];
                // For our size changed check, we keep mScreenRect.width() and mScreenRect.height()
                // in view local space.
                mLocation[0] = getWidth();
                mLocation[1] = getHeight();

                mScreenRect.set(mWindowSpaceLeft, mWindowSpaceTop,
                        mWindowSpaceLeft + mLocation[0], mWindowSpaceTop + mLocation[1]);

                if (mTranslator != null) {
                    mTranslator.translateRectInAppWindowToScreen(mScreenRect);
                }

                if (mSurfaceControl == null) {
                    return;
                }

                if (!isHardwareAccelerated() || !mRtHandlingPositionUpdates) {
                    try {
                        setParentSpaceRectangle(mScreenRect, -1);
                    } catch (Exception ex) {
                        Log.e(TAG, "Exception configuring surface", ex);
                    }
                }
            }
        }
    }
```

### SurfaceSession
相当于到surface flinger的连接
```
public final class SurfaceSession {
    // Note: This field is accessed by native code.
    private long mNativeClient; // SurfaceComposerClient*

    private static native long nativeCreate();
    private static native long nativeCreateScoped(long surfacePtr);
    private static native void nativeDestroy(long ptr);
    private static native void nativeKill(long ptr);

    /** Create a new connection with the surface flinger. */
    public SurfaceSession() {
        mNativeClient = nativeCreate();
    }

    public SurfaceSession(Surface root) {
        mNativeClient = nativeCreateScoped(root.mNativeObject);
    }

    /* no user serviceable parts here ... */
    @Override
    protected void finalize() throws Throwable {
        try {
            if (mNativeClient != 0) {
                nativeDestroy(mNativeClient);
            }
        } finally {
            super.finalize();
        }
    }

    /**
     * Forcibly detach native resources associated with this object.
     * Unlike destroy(), after this call any surfaces that were created
     * from the session will no longer work.
     */
    public void kill() {
        nativeKill(mNativeClient);
    }
}
```
在SurfaceView中使用了`SurfaceSession(Surface root)`，那么就看看`nativeCreateScoped`的具体实现
> 代码位于 /frameworks/base/core/jni/android_view_SurfaceSession.cpp

```
static jlong nativeCreateScoped(JNIEnv* env, jclass clazz, jlong surfaceObject) {
    Surface *parent = reinterpret_cast<Surface*>(surfaceObject);
    SurfaceComposerClient* client = new SurfaceComposerClient(parent->getIGraphicBufferProducer());
    client->incStrong((void*)nativeCreate);
    return reinterpret_cast<jlong>(client);
}
```
创建了SurfaceComposerClient对象，然后调用它的`incStrong`，然后引发`onFirstRef`调用（SurfaceComposerClient是RefBase子类，incStrong和onFirstRef都继承自RefBase）。
```
void SurfaceComposerClient::onFirstRef() {
    // 获取SurfacFlinger的代理BpSurfaceComposer
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != 0 && mStatus == NO_INIT) {
        auto rootProducer = mParent.promote();
        sp<ISurfaceComposerClient> conn;
        conn = (rootProducer != nullptr) ? sf->createScopedConnection(rootProducer) :
                sf->createConnection();
        if (conn != 0) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}
```
`createScopedConnection`会调用到SurfaceFlinger，返回BpSurfaceComposerClient，然后将client返回到Java层的SurfaceSession，赋值给mNativeClient。即mNativeClient对应C++的SurfaceComposerClient，SurfaceComposerClient内包含BpSurfaceComposerClient去和SurfaceFlinger跨进错交互。


### SurfaceControl
```
class SurfaceControlWithBackground extends SurfaceControl {
    SurfaceControl mBackgroundControl;
    private boolean mOpaque = true;
    public boolean mVisible = false;

    public SurfaceControlWithBackground(String name, boolean opaque, SurfaceControl.Builder b)
                   throws Exception {
        super(b.setName(name).build());

        mBackgroundControl = b.setName("Background for -" + name)
                .setFormat(OPAQUE)
                .setColorLayer(true)
                .build();
        mOpaque = opaque;
    }

    //......
}
```
SurfaceControl.Builder构造SurfaceControl：
```
public static class Builder {
    //.....

    public SurfaceControl build() {
        if (mWidth <= 0 || mHeight <= 0) {
            throw new IllegalArgumentException(
                    "width and height must be set");
        }
        return new SurfaceControl(mSession, mName, mWidth, mHeight, mFormat,
                mFlags, mParent, mWindowType, mOwnerUid);
    }

    //......
}
```
```
private SurfaceControl(SurfaceSession session, String name, int w, int h, int format, int flags,
        SurfaceControl parent, int windowType, int ownerUid)
                throws OutOfResourcesException, IllegalArgumentException {
    //......

    mName = name;
    mWidth = w;
    mHeight = h;
    mNativeObject = nativeCreate(session, name, w, h, format, flags,
        parent != null ? parent.mNativeObject : 0, windowType, ownerUid);
    if (mNativeObject == 0) {
        throw new OutOfResourcesException(
                "Couldn't allocate SurfaceControl native object");
    }

    mCloseGuard.open("release");
}
```
看到SurfaceControl的构造函数中，核心的是`nativeCreate`


> /frameworks/base/core/jni/android_view_SurfaceControl.cpp
```
static jlong nativeCreate(JNIEnv* env, jclass clazz, jobject sessionObj,
        jstring nameStr, jint w, jint h, jint format, jint flags, jlong parentObject,
        jint windowType, jint ownerUid) {
    ScopedUtfChars name(env, nameStr);
    // 通过SurfaceSession获取BpSurfaceComposerClient
    sp<SurfaceComposerClient> client(android_view_SurfaceSession_getClient(env, sessionObj));
    SurfaceControl *parent = reinterpret_cast<SurfaceControl*>(parentObject);
    sp<SurfaceControl> surface;
    // 通过BpSurfaceComposerClient创建SurfaceControl
    status_t err = client->createSurfaceChecked(
            String8(name.c_str()), w, h, format, &surface, flags, parent, windowType, ownerUid);
    //......

    surface->incStrong((void *)nativeCreate);
    return reinterpret_cast<jlong>(surface.get());
}
```
这里看到，创建了SurfaceControl，然后返回指针到Java层。C++层创建SurfaceControl（持有BpSurfaceComposerClient），会跨进程调用到SurfaceFlinger，SurfaceFlinger会创建Layer，SurfaceFlinger里的Layer对应的就是Java层的Surface。

### Surface
Surface的`copyFrom(SurfaceControl)`方法会调用`nativeGetFromSurfaceControl`创建Surface的native对象，然后将其指针保存到newNativeObject，这里直接看`nativeGetFromSurfaceControl`

> /frameworks/base/core/jni/android_view_Surface.cpp
```
static jlong nativeGetFromSurfaceControl(JNIEnv* env, jclass clazz,
        jlong surfaceControlNativeObj) {
    /*
     * This is used by the WindowManagerService just after constructing
     * a Surface and is necessary for returning the Surface reference to
     * the caller. At this point, we should only have a SurfaceControl.
     */

    sp<SurfaceControl> ctrl(reinterpret_cast<SurfaceControl *>(surfaceControlNativeObj));
    sp<Surface> surface(ctrl->getSurface());
    if (surface != NULL) {
        surface->incStrong(&sRefBaseOwner);
    }
    return reinterpret_cast<jlong>(surface.get());
}
```
重点是`getSurface()`

> /frameworks/native/libs/gui/SurfaceControl.cpp
```
sp<Surface> SurfaceControl::getSurface() const
{
    Mutex::Autolock _l(mLock);
    if (mSurfaceData == 0) {
        return generateSurfaceLocked();
    }
    return mSurfaceData;
}

sp<Surface> SurfaceControl::generateSurfaceLocked() const
{
    // This surface is always consumed by SurfaceFlinger, so the
    // producerControlledByApp value doesn't matter; using false.
    mSurfaceData = new Surface(mGraphicBufferProducer, false);

    return mSurfaceData;
}
```
这里创建了一个C++的Surface对象，Surface构造函数只是执行一些变量的初始化。C++的surface会持有mGraphicBufferProducer，用于跨进程操作位于SurfaceFlinger进程中的图形buffer

* Surface相当于是图像缓冲区的句柄
* SurfaceHolder接口是作为Controller角色存在。主要用来控制Surface大小、格式、监听Surface变化
* Surface是原始图像缓冲区（raw buffer）的一个句柄，而原始图像缓冲区是由屏幕图像合成器（screen compositor）管理的。 