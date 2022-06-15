下次研究invalidate，注意displayList。在[View的加载到绘制过程]有分析到，postInvalidateOnAnimation方法引发InvalidateOnAnimationRunnable的执行，对该view调用invalidate，似乎就是比直接调用invalidate延迟一帧？？？？


```
    public void invalidate() {
        invalidate(true);
    }
```
```
    public void invalidate(boolean invalidateCache) {
        //invalidateCache：该视图的绘图缓存是否也应无效。对于full invalidate，这通常是 
        //正确的，但是如果视图的内容或尺寸没有改变，则可以设置为false。
        invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
    }
```
```
    void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache,
            boolean fullInvalidate) {
        ......
        //不可见且没有运行动画就跳过
        if (skipInvalidate()) {
            return;
        }
        //根据mPrivateFlags来标记是否需要重绘，    假如View没有任何变化，那么就不需要重绘
        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
                || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
                || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
                || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
            //上面传入为true，表示需要全部重绘
            if (fullInvalidate) {
                mLastIsOpaque = isOpaque();//是否不透明
                mPrivateFlags &= ~PFLAG_DRAWN;//绘制完成标志位取反，表示需要绘制
            }
            ///添加标记，表示View已脏。
            mPrivateFlags |= PFLAG_DIRTY;
            //清除绘图缓存
            if (invalidateCache) {
                mPrivateFlags |= PFLAG_INVALIDATED;//表示已失效
                mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;//取反绘图缓存有效的标志位，表示缓存无效。
            }

            // 把脏的矩形范围传播放父视图
            final AttachInfo ai = mAttachInfo;
            final ViewParent p = mParent;
            if (p != null && ai != null && l < r && t < b) {
                final Rect damage = ai.mTmpInvalRect;
                //设置重绘区域(区域为当前View在父容器中的整个布局)
                damage.set(l, t, r, b);
                //调用了父容器 ViewGroup 的 invalidateChild()方法
                p.invalidateChild(this, damage);
            }
        ......
        }
    }
```
`ViewGroup` 的 `invalidateChild()`方法如下
```
    public final void invalidateChild(View child, final Rect dirty) {
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo != null && attachInfo.mHardwareAccelerated) {
            // 开启了硬件加速（后面未具体分析）
            onDescendantInvalidated(child, child);
            return;
        }

        // 未开启硬件加速

        //声明ViewParent 为自身
        ViewParent parent = this;
        if (attachInfo != null) {
            ......
            final boolean isOpaque = child.isOpaque() && !drawAnimation &&
                    child.getAnimation() == null && childMatrix.isIdentity();
            // 标记child dirty, 使用适当的 flag
            // 确保没有同时设置两个 flag
            //此时的child是调用invalidate的view，根据child是否Opaque，获得opaqueFlag
            int opaqueFlag = isOpaque ? PFLAG_DIRTY_OPAQUE : PFLAG_DIRTY;
            ......
            do {
                View view = null;
                if (parent instanceof View) {
                    //do-while循环，view每次循环都引用上一级的父容器，一直到顶级ViewGroup
                    view = (View) parent;
                }
                ......
                //经下面的if会设置每次循环的ViewGroup的mPrivateFlags（从调用invalidate 
                //的child的第一个父容器到顶级父容器）。若child的isOpaque为true（实心）
                //，则每个ViewGroup都会设置PFLAG_DIRTY_OPAQUE标志位。
                //结合后面的onDraw，可知只要child为实心，则它的父容器树均不会调用onDraw()，除了一些特殊情况。
                if (view != null) {
                    if ((view.mViewFlags & FADING_EDGE_MASK) != 0 &&
                            view.getSolidColor() == 0) {
                        opaqueFlag = PFLAG_DIRTY;
                    }
                    if ((view.mPrivateFlags & PFLAG_DIRTY_MASK) != PFLAG_DIRTY) {
                        view.mPrivateFlags = (view.mPrivateFlags & ~PFLAG_DIRTY_MASK) | opaqueFlag;
                    }
                }

                //invalidateChildInParent方法会返回父容器，从自身循环到parent==null
                //往上递归调用父容器的invalidateChildInParent，直到调用到ViewRootImpl的invalidateChildInParent
                parent = parent.invalidateChildInParent(location, dirty);
                ......
            } while (parent != null);
        }
    }
```
`invalidateChildInParent`方法如下
```
    public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
        if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID)) != 0) {
            // PFLAG_DRAWN 和 PFLAG_DRAWING_CACHE_VALID 只要有两个标志中的一个时，该方法就会返回 mParent
           ......
            return mParent;
        }
        return null;
    }
```
最终执行到`ViewRootImpl`的`invalidateChildInParent`方法
```
    @Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        //检查线程，这也是为什么invalidate一定要在主线程的原因
        checkThread();

        if (dirty == null) {
            invalidate();//有可能需要绘制整个窗口
            return null;
        } else if (dirty.isEmpty() && !mIsAnimating) {
            return null;
        }

        ``````
        
        invalidateRectOnScreen(dirty);

        return null;
    }
```
```
    //设置mDirty并执行View的工作流程
    private void invalidateRectOnScreen(Rect dirty) {
        final Rect localDirty = mDirty;
        if (!localDirty.isEmpty() && !localDirty.contains(dirty)) {
            mAttachInfo.mSetIgnoreDirtyState = true;
            mAttachInfo.mIgnoreDirtyState = true;
        }

        // 把新的脏区域矩形和当前的求并集，比如两个view都调用了invalidate，
        // 那么脏区域就会把两个view的脏区域合起来，最终就是得到了包含所有脏区域的矩形
        localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);    
        //在这里，mDirty的区域就变为方法中的dirty，即要重绘的脏区域

        ``````
        if (!mWillDrawSoon && (intersected || mIsAnimating)) {
            scheduleTraversals();//执行View的工作流程
        }
    }
```
&#160; &#160; &#160; &#160;也就是说`invalidate`方法会引起`scheduleTraversals()`方法的调用，但是此时并不会导致`measure`和`layout`的调用，只会调用`draw`。因为要引起`measure`和`layout`，需要设置`mLayoutRequested` 为`true`，但是`invalidate`这种情况不会设置。
&#160; &#160; &#160; &#160;下面`ViewRootImpl`的代码中可以看出。
```
//ViewRootImpl.java

    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();//检查是否在主线程
            mLayoutRequested = true;//mLayoutRequested 是否measure和layout布局。
            scheduleTraversals();
        }
    }

//可以看出 requestLayout() 和 invalidate() 都会调用到scheduleTraversals()方法，
//scheduleTraversals()又引发doTraversal---->performTraversals（View加载和绘制过程中有分析）

    private void performTraversals() {

        boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
        if (layoutRequested) {
            //此方法会调用performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            measureHierarchy(```);
        }


        final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
        if (didLayout) {
            performLayout(lp, mWidth, mHeight);//layout
        }


        boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
        if (!cancelDraw && !newSurface) {
            performDraw();//draw
        }
    } 
```
`View`的`Draw()`方法如下：
```
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

        // Step 1, draw the background, if needed
        int saveCount;
        //一般情况，invalidateChild中的do-while循环对每个ViewGroup设置了PFLAG_DIRTY_OPAQUE 
        //所以ViewGroup不会进入下面if（如果调用setBackground()，应该会修改标志位）
        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // 同上，ViewGroup不会进入下面if，不会调用onDraw()
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we're done...
            return;
        }

        /*
         * Here we do the full fledged routine...
         * (this is an uncommon case where speed matters less,
         * this is why we repeat some of the tests that have been
         * done above)
         */
         ......省略下面非一般情况
    }
```
为了保证绘制的效率，控件树仅对需要重绘的区域进行绘制。这部分区域成为”脏区域”Dirty Area。

当一个控件的内容发生变化而需要重绘时，它会通过`View.invalidate()`方法将其需要重绘的区域沿着控件树自下而上的交给`ViewRootImpl`，并保存在`ViewRootImpl`的`mDirty`成员中，最后通过`scheduleTraversals()`引发一次遍历，进而进行重绘工作，这样就可以保证仅位于`mDirty`所描述的区域得到重绘，避免了不必要的开销。

View的`isOpaque()`方法返回值表示此控件是否为”实心”的，所谓”实心”控件，是指在`onDraw()`方法中能够保证此控件的所有区域都会被其所绘制的内容完全覆盖。对于”实心”控件来说，背景和子元素（如果有的话）是被其`onDraw()`的内容完全遮住的，因此便可跳过遮挡内容的绘制工作从而提升效率。

简单来说透过此控件所属的区域无法看到此控件下的内容，也就是既没有半透明也没有空缺的部分。因为自定义`ViewGroup`控件默认是”实心”控件，所以默认不会调用`drawBackground()`和`onDraw()`方法，因为一旦`ViewGroup`的`onDraw()`方法，那么就会覆盖住它的子元素。但是我们仍然可以通过调用`setWillNotDraw(false)`和`setBackground()`方法来开启`ViewGroup`的`onDraw()`功能。