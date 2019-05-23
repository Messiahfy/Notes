在`View.measure(int,int)`的方法里，仅当给与的MeasureSpec发生变化时，或要求强制重新布局时（`if (forceLayout || needsLayout)`），才会调用`onMeasure(int,int)`。

**强制重新布局** ： 控件树中的一个子控件内容发生变化时，需要重新测量和布局的情况，在这种情况下，这个子控件的父控件（以及父控件的父控件）所提供的MeasureSpec必定与上次测量时的值相同，因而导致从ViewRootImpl到这个控件的路径上，父控件的onMeasure()方法无法得到执行，进而导致子控件无法重新测量其布局和尺寸。

**解决途径** : 因此，当子控件因内容发生变化时，从子控件到父控件回溯到ViewRootImpl，并依次调用父控件的requestLayout()方法。这个方法会在mPrivateFlags中加入标记PFLAG_FORCE_LAYOUT，从而使得这些父控件的measure()方法得以顺利执行，进而这个子控件有机会进行重新布局与测量。这便是强制重新布局的意义所在。

`View`的`RequestLayout()`代码如下：
```
    public void requestLayout() {
        if (mMeasureCache != null) mMeasureCache.clear();
        ......
        //设置了强制重新布局的标志位，在meausre时检测
        mPrivateFlags |= PFLAG_FORCE_LAYOUT;
        mPrivateFlags |= PFLAG_INVALIDATED;
        // 父类不为空&&父类没有请求重新布局(是否有PFLAG_FORCE_LAYOUT标志)
        //这样同一个父容器的多个子View同时调用requestLayout()就不会增加开销
        if (mParent != null && !mParent.isLayoutRequested()) {
            mParent.requestLayout();
        }
        ......
    }
```
会一直执行到`ViewRootImpl`的`requestLayout() `方法
```
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();//也是执行scheduleTraversals()方法
        }
    }
```
设置了`mLayoutRequested =true`标识，所以在`performTraversals()`中调用`performMeasure()`，`performLayout()`，但是由于没有设置`mDirty`，所以不会走`performDraw()`流程。

但也并非绝对不会导致`onDraw()`的调用，在`View`的`layout()`方法里，首先通过 `setFrame()`（`setOpticalFrame()`也走`setFrame()`）将l、t、r、b分别设置到mLeft、mTop、mRight、和mBottom，这样就可以确定子View在父容器的位置了，这些位置是相对父容器的。
```
//View -->  layout()
    protected boolean setFrame(int left, int top, int right, int bottom) {
        boolean changed = false;


        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
            //布局坐标改变了
            changed = true;

            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);

            // Invalidate our old position
            invalidate(sizeChanged);//调用invalidate重新绘制视图


            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);
            }

            ``````
        }
        return changed;
    }
```
此时可知，如果layout布局有变化，还是会调用invalidate重绘自身。
![requestLayout()流程图](https://upload-images.jianshu.io/upload_images/3468445-e4818921905499f6.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
