## SnapHelper
SnapHelper可以在滑动RecyclerView松手时，不是按照惯性自然滑动和停止，而是可以调整滚动位置，例如滑动停止时让item居中，或者翻页效果。SnapHelper是一个继承OnFlingListener的抽象类，需要继承它重写一些方法才可以使用，官方有LinearSnapHelper和PagerSnapHelper来实现不同的效果，ViewPager2中因为使用了RecyclerView，所以也用到了SnapHelper。

### 代码原理
跟着SnapHelper的代码流程，来学习它的原理

#### attachToRecyclerView
在使用SnapHelper的时候，只需要创建它然后调用`attachToRecyclerView`函数即可，例如子类LinearSnapHelper：
```
LinearSnapHelper().attachToRecyclerView(recyclerView)

public void attachToRecyclerView(@Nullable RecyclerView recyclerView)
        throws IllegalStateException {
    if (mRecyclerView == recyclerView) {
        return; // 已经attach，则跳过
    }
    if (mRecyclerView != null) {
        destroyCallbacks(); // 现在attach了一个新的recyclerView，则把原本的回调移除
    }
    mRecyclerView = recyclerView;
    if (mRecyclerView != null) {
        setupCallbacks(); // 设置OnScrollListener和OnFlingListener
        mGravityScroller = new Scroller(mRecyclerView.getContext(),
                new DecelerateInterpolator());
        snapToTargetExistingView(); // 完成对齐滚动
    }
}
```

`snapToTargetExistingView`
```
void snapToTargetExistingView() {
    if (mRecyclerView == null) {
        return;
    }
    RecyclerView.LayoutManager layoutManager = mRecyclerView.getLayoutManager();
    if (layoutManager == null) {
        return;
    }
    // 找到 snapView
    View snapView = findSnapView(layoutManager);
    if (snapView == null) {
        return;
    }
    // 计算snapView需要滚动的距离
    int[] snapDistance = calculateDistanceToFinalSnap(layoutManager, snapView);
    if (snapDistance[0] != 0 || snapDistance[1] != 0) {
        // 执行滚动
        mRecyclerView.smoothScrollBy(snapDistance[0], snapDistance[1]);
    }
}
```

#### 回调

给RecyclerView设置了这个OnScrollListener，作用就是在滚动结束后，执行snapToTargetExistingView()
```
private final RecyclerView.OnScrollListener mScrollListener =
        new RecyclerView.OnScrollListener() {
            boolean mScrolled = false;

            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                if (newState == RecyclerView.SCROLL_STATE_IDLE && mScrolled) {
                    mScrolled = false;
                    snapToTargetExistingView(); // 执行
                }
            }
            @Override
            public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                if (dx != 0 || dy != 0) {
                    mScrolled = true;
                }
            }
        };
```

另外就是设置OnFlingListener，这个是自身实现：
```
@Override
public boolean onFling(int velocityX, int velocityY) {
    RecyclerView.LayoutManager layoutManager = mRecyclerView.getLayoutManager();
    if (layoutManager == null) {
        return false;
    }
    RecyclerView.Adapter adapter = mRecyclerView.getAdapter();
    if (adapter == null) {
        return false;
    }
    int minFlingVelocity = mRecyclerView.getMinFlingVelocity();
    return (Math.abs(velocityY) > minFlingVelocity || Math.abs(velocityX) > minFlingVelocity)
            && snapFromFling(layoutManager, velocityX, velocityY);
}
```
重点就是在达到了fling的最小速度时，执行`snapFromFling`

```
private boolean snapFromFling(@NonNull RecyclerView.LayoutManager layoutManager, int velocityX,
        int velocityY) {
    // layoutManager必须实现ScrollVectorProvider
    if (!(layoutManager instanceof RecyclerView.SmoothScroller.ScrollVectorProvider)) {
        return false;
    }

    // 创建SmoothScroller
    RecyclerView.SmoothScroller smoothScroller = createScroller(layoutManager);
    if (smoothScroller == null) {
        return false;
    }

    // 找到目标SnapPosition
    int targetPosition = findTargetSnapPosition(layoutManager, velocityX, velocityY);
    if (targetPosition == RecyclerView.NO_POSITION) {
        return false;
    }

    // 设置目标SnapPosition
    smoothScroller.setTargetPosition(targetPosition);
    layoutManager.startSmoothScroll(smoothScroller); // 使用layoutManager启动SmoothScroll，滚动到目标位置
    return true;
}
```
使用`setTargetPosition()`设置了smoothScroller后，只能滚动到目标位置和RecyclerView边缘对齐，而如果要做一些调整，就要在SmoothScroller的`onTargetFound`（会在目标view布局时回调）回调时，再利用`calculateDistanceToFinalSnap`计算距离来滚动调整。


> 这里的OnScrollListener实现是在IDEL时才执行snap滚动，而OnFlingListener用于松手时还要继续滚动的情况，所以两者不会同时作用

```
// 该方法会根据触发Fling操作的速率（参数velocityX和参数velocityY）来找到RecyclerView需要滚动到哪个位置，该位置对应的ItemView就是那个需要进行对齐的列表项。我们把这个位置称为targetSnapPosition，对应的View称为targetSnapView。如果找不到targetSnapPosition，就返回RecyclerView.NO_POSITION。（这个函数仅会在fling的情况下使用，fling的情况会滚动到目前还未展示的item位置，所以要计算目标位置）
abstract int findTargetSnapPosition(RecyclerView.LayoutManager layoutManager, int velocityX, int velocityY);

// 该方法会找到当前layoutManager上最接近对齐位置的那个view，该view称为SanpView，对应的position称为SnapPosition。如果返回null，就表示没有需要对齐的View，也就不会做滚动对齐调整。
abstract View findSnapView(RecyclerView.LayoutManager layoutManager);

// 计算SnapView到目标位置的距离。例如LinearSnapHelper中，最接近居中的就是SnapView，目标位置就是居中
abstract int[] calculateDistanceToFinalSnap(@NonNull RecyclerView.LayoutManager layoutManager, @NonNull View targetView);

RecyclerView.SmoothScroller createScroller(RecyclerView.LayoutManager layoutManager)
```