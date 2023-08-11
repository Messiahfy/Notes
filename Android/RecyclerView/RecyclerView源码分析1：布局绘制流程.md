* RecyclerViewDataObserver 数据观察器
* Recycler View循环复用系统，核心部件
* SavedState RecyclerView状态
* AdapterHelper 适配器更新
* ChildHelper 管理子View
* ViewInfoStore 存储子VIEW的动画信息
* Adapter 数据适配器
* LayoutManager 负责子View的布局，核心部件
* ItemAnimator Item动画
* ViewFlinger 快速滑动管理
* NestedScrollingChildHelper 管理子View嵌套滑动
## 构造函数
```
public RecyclerView(Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        ......
        setScrollContainer(true);//表示RecyclerView是滚动容器，和软键盘弹出改变Window大小有关
        setFocusableInTouchMode(true);//可接受焦点focuse

        final ViewConfiguration vc = ViewConfiguration.get(context);
        mTouchSlop = vc.getScaledTouchSlop();//设置判定为滑动的最小dp距离
        mScaledHorizontalScrollFactor =
                ViewConfigurationCompat.getScaledHorizontalScrollFactor(vc, context);
        mScaledVerticalScrollFactor =
                ViewConfigurationCompat.getScaledVerticalScrollFactor(vc, context);
        //启动fling的最小速度，即抬手时大于等于这个速度才会有逐渐减速的效果
        mMinFlingVelocity = vc.getScaledMinimumFlingVelocity();
        mMaxFlingVelocity = vc.getScaledMaximumFlingVelocity();
        //表示自身不需要draw，有一定的优化作用
        setWillNotDraw(getOverScrollMode() == View.OVER_SCROLL_NEVER);
        //对默认动画设置监听器
        mItemAnimator.setListener(mItemAnimatorListener);
        initAdapterManager();//构造AdapterHelper
        initChildrenHelper();//构造ChildHelper
        ......
        // Create the layoutManager if specified.

        boolean nestedScrollingEnabled = true;

        if (attrs != null) {
            int defStyleRes = 0;
            TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.RecyclerView,
                    defStyle, defStyleRes);
            String layoutManagerName = a.getString(R.styleable.RecyclerView_layoutManager);
            int descendantFocusability = a.getInt(
                    R.styleable.RecyclerView_android_descendantFocusability, -1);
            if (descendantFocusability == -1) {
                setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            }
            //设置快速滑动栏和滑块的drawable，相当于浏览器的右侧拖动快速滑动
            mEnableFastScroller = a.getBoolean(R.styleable.RecyclerView_fastScrollEnabled, false);
            if (mEnableFastScroller) {
                StateListDrawable verticalThumbDrawable = (StateListDrawable) a
                        .getDrawable(R.styleable.RecyclerView_fastScrollVerticalThumbDrawable);//滑动块
                Drawable verticalTrackDrawable = a
                        .getDrawable(R.styleable.RecyclerView_fastScrollVerticalTrackDrawable);//滑动背景
                StateListDrawable horizontalThumbDrawable = (StateListDrawable) a
                        .getDrawable(R.styleable.RecyclerView_fastScrollHorizontalThumbDrawable);
                Drawable horizontalTrackDrawable = a
                        .getDrawable(R.styleable.RecyclerView_fastScrollHorizontalTrackDrawable);
                initFastScroller(verticalThumbDrawable, verticalTrackDrawable,
                        horizontalThumbDrawable, horizontalTrackDrawable);
            }
            a.recycle();
            //如果在xml中设置了LayoutManager，就使用反射来构造
            createLayoutManager(context, layoutManagerName, attrs, defStyle, defStyleRes);

            if (Build.VERSION.SDK_INT >= 21) {//嵌套滑动相关
                a = context.obtainStyledAttributes(attrs, NESTED_SCROLLING_ATTRS,
                        defStyle, defStyleRes);
                nestedScrollingEnabled = a.getBoolean(0, true);
                a.recycle();
            }
        } else {
            setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        }

        // 设置是否启用嵌套滑动，默认启用
        setNestedScrollingEnabled(nestedScrollingEnabled);
    }
```
## setLayoutManager
如果之前已设LayoutManager，就先移除所有view并回收，然后清空`Recycler`；如果之前没有就只是清空`Recycler`。`mLayout`引用新的`LayoutManager`，并调用`requestLayout()`
```
    public void setLayoutManager(LayoutManager layout) {
        if (layout == mLayout) {
            return;
        }
        stopScroll();
        // TODO We should do this switch a dispatchLayout pass and animate children. There is a good
        // chance that LayoutManagers will re-use views.
        if (mLayout != null) {
            // 结束所有动画
            if (mItemAnimator != null) {
                mItemAnimator.endAnimations();
            }
            mLayout.removeAndRecycleAllViews(mRecycler);//移除所有View并回收
            mLayout.removeAndRecycleScrapInt(mRecycler);
            mRecycler.clear();

            if (mIsAttached) {
                mLayout.dispatchDetachedFromWindow(this, mRecycler);
            }
            mLayout.setRecyclerView(null);
            mLayout = null;
        } else {
            mRecycler.clear();
        }
        // this is just a defensive measure for faulty item animators.
        mChildHelper.removeAllViewsUnfiltered();
        mLayout = layout;
        if (layout != null) {
            if (layout.mRecyclerView != null) {
                throw new IllegalArgumentException("LayoutManager " + layout
                        + " is already attached to a RecyclerView:"
                        + layout.mRecyclerView.exceptionLabel());
            }
            mLayout.setRecyclerView(this);
            if (mIsAttached) {
                mLayout.dispatchAttachedToWindow(this);
            }
        }
        mRecycler.updateViewCacheSize();
        requestLayout();
    }
```
## setAdapter
```
    public void setAdapter(Adapter adapter) {
        // bail out if layout is frozen
        setLayoutFrozen(false);
        setAdapterInternal(adapter, false, true);//LayoutManager移除view，Recycler回收，设置新的adapter
        processDataSetCompletelyChanged(false);
        requestLayout();//也会引发绘制
    }
```
--------------------------------------------------
> 设置`LayoutManager`和`Adapter`会引发绘制流程，修改数据并`notify`也会引发绘制流程
## onMeasure
RecyclerView中有一个mState，它的mLayoutStep字段用于标识当前需要执行哪一步布局：
* `STEP_START`：需要执行dispatchLayoutStep1()，执行后状态为STEP_LAYOUT
* `STEP_LAYOUT`：需要执行dispatchLayoutStep2()，执行后状态为STEP_ANIMATIONS
* `STEP_ANIMATIONS`：需要执行dispatchLayoutStep3()，执行后状态又变为STEP_START
```
    @Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        // 第一种情况，没有设置LayoutManager
        if (mLayout == null) {
            //如果没有设置layoutManager，调用default方法，设置宽高
            defaultOnMeasure(widthSpec, heightSpec);
            return;
        }

        // 第二种情况，LayoutManager开启自动测量，一般都是这种
        if (mLayout.isAutoMeasureEnabled()) { //LinearLayoutManager返回true
            final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
            //内部也是调用defaultOnMeasure设置自身默认宽高
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

            final boolean measureSpecModeIsExactly =
                    widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
            if (measureSpecModeIsExactly || mAdapter == null) {
                return;//如果宽高都是确定值，或者adapter为空，就直接返回，然后走Layout流程
            }
            //如果宽高不是确定值，就要onMeasure中执行step1和step2
            if (mState.mLayoutStep == State.STEP_START) {
                dispatchLayoutStep1();//执行完后变State.STEP_LAYOUT
            }
            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();//真正执行LayoutManager布局的地方
            //执行完后是State.STEP_ANIMATIONS

            // 执行step1和step2后，使用得到children的宽高来设置RecyclerView长宽的测量值
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            // if RecyclerView has non-exact width and height and if there is at least one child
            // which also has non-exact width & height, we have to re-measure.
            if (mLayout.shouldMeasureTwice()) {
                mLayout.setMeasureSpecs(
                        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
                mState.mIsMeasuring = true;
                dispatchLayoutStep2();
                // now we can get the width and height from the children.
                mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            }
        } else {
            ......一般是上面的分支
        }
    }
```
> 如果`RecyclerVeiw`的宽高不确定，就在`onMeasure`中执行`dispatchLayoutStep1()`和`dispatchLayoutStep2()`，`onMeasure`中执行`1`和`2`的话，`onLayout`中就不再执行<br>
否则，如果确定，`onMeasure`中不执行`1`和`2`，而在`onLayout`中执行。

## onLayout
```
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();//主要是此方法
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}
```

```
void dispatchLayout() {
        ......
        mState.mIsMeasuring = false;
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            // First 2 steps are done in onMeasure but looks like we have to run again due to
            // changed size.
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            // always make sure we sync them (to ensure mode is exact)
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3();
    }
```
> 如果在`onMeasure`中已经完成了`step1`与`step2`，则只会执行`step3`，否则三步会依次触发。

------------------------------------------

**不论onMeasure和onlayout中是什么情况，`step1`、`step2`和`step3`都会依次执行，所以下面分析这三个方法**
## dispatchLayoutStep1
1. 处理`Adapter`的更新
2. 决定要执行的动画
3. 保存当前`Views`的信息
4. 如果有必要的话再进行上一布局操作，并保存它的信息
```
    private void dispatchLayoutStep1() {
        mState.assertLayoutStep(State.STEP_START);
        fillRemainingScrollValues(mState);
        mState.mIsMeasuring = false;
        startInterceptRequestLayout();//拦截RequestLayout，防止冗余调用
        mViewInfoStore.clear();
        onEnterLayoutOrScroll();
        processAdapterUpdatesAndSetAnimationFlags();//使用adapter更新和计算想运行的动画类型
        saveFocusInfo();
        mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
        mItemsAddedOrRemoved = mItemsChanged = false;
        mState.mInPreLayout = mState.mRunPredictiveAnimations;
        mState.mItemCount = mAdapter.getItemCount();
        findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);

        if (mState.mRunSimpleAnimations) {
            // 这里需要运行动画的情况下进行的，主要做的事情就是找出那些要需要进
            // 行上一布局操作的ViewHolder，并且保存它们的边界信息。如果有更新操作(这个更新
            // 指的是内容的更新，不是插入删除的这种更新)，然后保存这些更新的ViewHolder
            ......
        }
        if (mState.mRunPredictiveAnimations) {
            ......//需要在布局结束之后运行动画的情况下执行
            clearOldPositions();
        } else {
            clearOldPositions();
        }
        onExitLayoutOrScroll();
        stopInterceptRequestLayout(false);
        mState.mLayoutStep = State.STEP_LAYOUT;
    }
```

## dispatchLayoutStep2
执行实际布局
```
    private void dispatchLayoutStep2() {
        ......
        mState.mItemCount = mAdapter.getItemCount();//Adapter中重写的getItemCount方法
        ......
        mState.mInPreLayout = false;
        //此方法执行实际布局，把布局操作交给LayoutManager来完成
        mLayout.onLayoutChildren(mRecycler, mState);
        ......
    }
```
## dispatchLayoutStep3
设置状态，触发动画，清理无用信息
```
private void dispatchLayoutStep3() {
    …… // 设置状态
    if (mState.mRunSimpleAnimations) {
        …… // 需要动画的情况。找出ViewHolder现在的位置，并且处理改变动画。最后触发动画。
    }

    …… // 清除状态和清除无用的信息
    mLayout.onLayoutCompleted(mState); // 给LayoutManager的布局完成的回调
    …… // 清除状体和清楚无用的信息，最后在恢复一些信息信息，比如焦点。
}
```
---------------------------------------------
从上面的几个步骤可以看出，很多是和动画、保存信息等相关的操作，这里先不管动画之类的内容，先聚焦于`LayoutManager`的`onLayoutChildren`方法，此方法是实际完成布局的方法。

## 以LinearManager的onLayoutChildren方法为例
```
    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        // 布局算法：
        // 1) 通过检查子项和其他变量，找到锚点坐标和锚点项目位置。
        // 2) fill towards start, stacking from bottom
        // 3) fill towards end, stacking from top
        // 4) scroll to fulfill requirements like stack from bottom.
        // create layout state
        
        if (mPendingSavedState != null || mPendingScrollPosition != NO_POSITION) {
            if (state.getItemCount() == 0) {
                removeAndRecycleAllViews(recycler);
                return;//children数量为0就清空，并返回
            }
        }
        if (mPendingSavedState != null && mPendingSavedState.hasValidAnchor()) {
            mPendingScrollPosition = mPendingSavedState.mAnchorPosition;
        }

        ensureLayoutState();
        mLayoutState.mRecycle = false;
        // 处理布局方向
        resolveShouldLayoutReverse();

        final View focused = getFocusedChild();
        if (!mAnchorInfo.mValid || mPendingScrollPosition != NO_POSITION
                || mPendingSavedState != null) {
            mAnchorInfo.reset();
            mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
            // 计算锚点的位置和坐标
            updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
            mAnchorInfo.mValid = true;
        } else if (focused != null && (mOrientationHelper.getDecoratedStart(focused)
                        >= mOrientationHelper.getEndAfterPadding()
                || mOrientationHelper.getDecoratedEnd(focused)
                <= mOrientationHelper.getStartAfterPadding())) {
            ......
        }
        ......
        
        onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
        detachAndScrapAttachedViews(recycler);
        mLayoutState.mInfinite = resolveIsInfinite();
        mLayoutState.mIsPreLayout = state.isPreLayout();
        if (mAnchorInfo.mLayoutFromEnd) {
            // fill towards start
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForStart;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
            final int firstElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForEnd += mLayoutState.mAvailable;
            }
            // fill towards end
            updateLayoutStateToFillEnd(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForEnd;
            mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;

            if (mLayoutState.mAvailable > 0) {
                // end could not consume all. add more items towards start
                extraForStart = mLayoutState.mAvailable;
                updateLayoutStateToFillStart(firstElement, startOffset);
                mLayoutState.mExtraFillSpace = extraForStart;
                fill(recycler, mLayoutState, state, false);
                startOffset = mLayoutState.mOffset;
            }
        } else {
            // fill towards end
            updateLayoutStateToFillEnd(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForEnd;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
            final int lastElement = mLayoutState.mCurrentPosition;
            if (mLayoutState.mAvailable > 0) {
                extraForStart += mLayoutState.mAvailable;
            }
            // fill towards start
            updateLayoutStateToFillStart(mAnchorInfo);
            mLayoutState.mExtraFillSpace = extraForStart;
            mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;

            if (mLayoutState.mAvailable > 0) {
                extraForEnd = mLayoutState.mAvailable;
                // start could not consume all it should. add more items towards end
                updateLayoutStateToFillEnd(lastElement, endOffset);
                mLayoutState.mExtraFillSpace = extraForEnd;
                fill(recycler, mLayoutState, state, false);
                endOffset = mLayoutState.mOffset;
            }
        }

        ......
        layoutForPredictiveAnimations(recycler, state, startOffset, endOffset);
        if (!state.isPreLayout()) {
            mOrientationHelper.onLayoutComplete();
        } else {
            mAnchorInfo.reset();
        }
        mLastStackFromEnd = mStackFromEnd;
        if (DEBUG) {
            validateChildOrder();
        }
    }
```
找到锚点，然后使用fill方法，对两个方向添加view
## fill方法
```
    int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
            RecyclerView.State state, boolean stopOnFocusable) {
        ......
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            ......//只要有剩余空间且在adapter中还有item没有绘制
            layoutChunk(recycler, state, layoutState, layoutChunkResult);
            ......
        }
        return start - layoutState.mAvailable;
    }
```
关键是`layoutChunk`方法

## layoutChunk方法
```
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
        View view = layoutState.next(recycler);//获得下一个要布局的元素的view
        if (view == null) {
            ......
            return;
        }
        LayoutParams params = (LayoutParams) view.getLayoutParams();
        if (layoutState.mScrapList == null) {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                addView(view);
            } else {
                addView(view, 0);
            }
        } else {
            ......
        }
        measureChildWithMargins(view, 0, 0);//测量该子view
        ......
        // 布局，调用child的layout方法
        layoutDecoratedWithMargins(view, left, top, right, bottom);
        ......
    }
```

循环调用`layoutChunk`方法完成了把`children`添加到了`RecyclerView`中作为子`view`。

`layoutState.next(recycler)`方法内部是回收复用或者创建新的`Holder`的相关代码，获取到`Holder`还会调用`tryBindViewHolderByDeadline`方法，引发调用`Adapter`的`onBindViewHolder`方法，最后把`Holder`的`View`返回。
> `layoutState.next(recycler)`相关具体内容在下一节**回收复用**中分析

------------------------------------------------------ 

## 绘制
关于绘制，`RecyclerView`的`draw`方法会完成`ItemDecoration`的绘制，而且会调用`super.draw(c)`-->`ViewGroup.dispatchDraw(canvas)`-->循环调用`drawChild`-->子`view`的绘制