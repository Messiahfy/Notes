## 1.概述
`scrollTo()`或`scrollBy()`方法不能移动自身，而是移动自身的内容，自身位置并不会改变，所以响应事件的位置也不会改变。这样看来`scrollTo()`或`scrollBy()`方法并不好用，但其实当对一个`ViewGroup`类型例如`ScrollView`这类的容器调用`scrollTo()`或`scrollBy()`方法，会移动它里面的内容，移动的内容的位置会发生改变，内容的响应事件位置也会改变，所以一般将`scrollTo`或`scrollBy`方法用于可以滑动内容的容器视图。

但仅仅使用`scrollTo()`或`scrollBy()`方法，是瞬间把内容滑到某位置。而要实现渐进式滑动内容，一般使用**Scroller**。
> **要注意的是**：使用`Scroller`并不是只能用于`scrollX`、`scrollY`，我们只是把值传入，然后`Scroller`帮我们计算当前值，所以也可以传入`left`等坐标，用来计算自身位置，而不是仅用于移动内容。
**还可以修改默认插值器**

## 2.使用方式
在需要滑动内容的视图中，添加如下代码：
要注意的是，使用`scroll`系列方法，传入负值才是把内容往X的右和Y的下方移动。

```
Scroller scroller = new Scroller(getContext());

//滑动到指定位置（或者改写成滑动距离，看需求）
public void smoothScrollTo(int destX, int destY) {
    int scrollX = getScrollX();
    int scrollY = getScrollY();
    int deltaX = destX - scrollX;
    int deltaY = destY - scrollY;
    scroller.startScroll(scrollX, scrollY, deltaX, deltaY, 1000);
    invalidate();
}

@Override
public void computeScroll() {
    if (scroller != null) {
        if (scroller.computeScrollOffset()) {
            //内部会调用invalidate之类的方法，不断重绘，直到if语句false
            scrollTo(scroller.getCurrX(), scroller.getCurrY());
            //如果不是调用 scrollTo() ，则必须保证调用 invalidate 相关方法
            //不论是调用的方法已调用，还是自行调用
            invalidate();//scrollTo内部已调用，这里可以不调用
        }
    }
}
```

## 3.源码分析
#### 3.1 
首先查看`startScroll`方法，只是保存传入的参数。
```
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
    mMode = SCROLL_MODE;
    mFinished = false;
    mDuration = duration;
    mStartTime = AnimationUtils.currentAnimationTimeMillis();
    mStartX = startX;
    mStartY = startY;
    mFinalX = startX + dx;
    mFinalY = startY + dy;
    mDeltaX = dx;
    mDeltaY = dy;
    mDurationReciprocal = 1.0f / (float) mDuration;
}
```
&emsp;&emsp;`startX`和`startY`表示滑动的起点，`dx`和`dy`表示要滑动的距离，`duration`表示滑动时间。可以看出`startScroll`方法并没有让视图的内容滑动起来，而关键是在调用`startScroll`方法之后调用的`invalidate()`方法。
&emsp;&emsp;`invalidate()`方法会引发视图重绘，在`View`的`draw`方法中会去调用`computeScroll()`方法，这个方法在`View`中是一个空方法，需要我们自己去实现，在前面使用方式中就实现了该方法。在`computeScroll()`中从`Scroller`中获取当前的`scrollX`和`scrollY`，调用`scrollTo()`方法实现滑动，`scrollTo()`中又会调用`invalidate()`方法，然后重复这个过程直到`scroller.computeScrollOffset()`返回`false`。

#### 3.2
看`Scroller`中的`computeScrollOffset()`如何实现：
```
/**
 * 当想知道新的位置时调用此方法。如果返回true，则动画未结束。
 */ 
public boolean computeScrollOffset() {
    if (mFinished) {
        return false;
    }
    //获取当前时间减去开始时间的值，即经过的时间
    int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    //经过的时间小于duration才继续计算
    if (timePassed < mDuration) {
        switch (mMode) {
        case SCROLL_MODE:
            //根据经过的时间和插值器计算 当前值/(结束值 - 开始值) 这个比值
            final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
            //用这个比值来计算当前值
            mCurrX = mStartX + Math.round(x * mDeltaX);
            mCurrY = mStartY + Math.round(x * mDeltaY);
            break;
        case FLING_MODE:
            ......//省略fling模式
        }
    }
    else {//经过的时间大于或等于duration，就结束计算
        mCurrX = mFinalX;
        mCurrY = mFinalY;
        mFinished = true;
    }
    return true;
}
```
注释中描述了代码的逻辑，关键就是根据当前时间来计算当前的值

## 4.常用API
| 方法 | 描述 |
|--|--|
| `abortAnimation()` | 将`mFinished`设为`true`，并把`mCurrX`和`mCurrY`设为结束值。<br>按**使用方式**的写法，`mFinished`设为`true`后，`computeScrollOffset()`返回`false`，就不会再进入`if`语句去调用`getCurrX()`和`Y`，所以修改了`mCurrX`和`mCurrY`并没有影响，其他情况需具体考虑`mCurrX`和`mCurrY`的影响 |
| `isFinished()` | 返回`mFinished`这个布尔值 |
| `forceFinished(boolean finished)` | 对`mFinished`赋值。`computeScrollOffset`方法中会判断如果`mFinished` 为`true`，就返回`false`停止计算|
| `getDuration()` | 返回中时长 |
| `getDuration()` | 返回中时长 |
| `getCurrX()`、`getCurrY()` | 返回当前的X和Y的值 |
| `getStartX()`、`getStartY()` | 返回开始的X和Y的值 |
| `getFinalX()`、`getFinalY()`，和对应的setter | 返回结束的X和Y的值 |
| `getCurrVelocity()` | 返回当前速度（fling模式） |

## 5.fling
`Scroller`还可以使用`fling`方法来完成惯性滑动，即滑动抬手（带有滑动速度）后不断减速的情况。一般需要结合`VelocityTracker`获取速度来使用，还可以配合`GestureDectector`检测fling手势。

先看`fling`方法和`computeScrollOffset`方法源码。
```
/**
 * 滑动的距离依赖于初始速度，越快越远
 * 
 * @param startX         开始的mScrollX位置
 * @param startY         开始的mScrollY位置
 * @param velocityX      初始X轴速率，单位：像素/秒
 * @param velocityY      初始Y轴速率，单位：像素/秒
 * @param minX           最小X值，不能超过此位置
 * @param maxX           最大X值，不能超过此位置
 * @param minY           最小Y值，不能超过此位置
 * @param maxY           最大Y值，不能超过此位置
 */
public void fling(int startX, int startY, int velocityX, int velocityY,
            int minX, int maxX, int minY, int maxY) {
    // 如果当前正在滑动或 fling ，当前速度和设置的速度同向的话，就加起来（速度更快）
    if (mFlywheel && !mFinished) {
        float oldVel = getCurrVelocity();

        float dx = (float) (mFinalX - mStartX);
        float dy = (float) (mFinalY - mStartY);
        float hyp = (float) Math.hypot(dx, dy);

        float ndx = dx / hyp;
        float ndy = dy / hyp;
        //把当前速度分解为X和Y的速度
        float oldVelocityX = ndx * oldVel;
        float oldVelocityY = ndy * oldVel;
        //速度同向，就加起来
        if (Math.signum(velocityX) == Math.signum(oldVelocityX) &&
                Math.signum(velocityY) == Math.signum(oldVelocityY)) {
            velocityX += oldVelocityX;
            velocityY += oldVelocityY;
        }
    }

    mMode = FLING_MODE;//模式设为fling
    mFinished = false;//结束标识设为fasle
    //合成斜边速度，即未分解的实际滑动速度
    float velocity = (float) Math.hypot(velocityX, velocityY);
     
    mVelocity = velocity;
    mDuration = getSplineFlingDuration(velocity);//根据速度算出总时间
    mStartTime = AnimationUtils.currentAnimationTimeMillis();//当前时间
    mStartX = startX;//起始位置
    mStartY = startY;

    float coeffX = velocity == 0 ? 1.0f : velocityX / velocity;
    float coeffY = velocity == 0 ? 1.0f : velocityY / velocity;

    double totalDistance = getSplineFlingDistance(velocity);
    mDistance = (int) (totalDistance * Math.signum(velocity));
    //赋值最大最小的X和Y
    mMinX = minX;
    mMaxX = maxX;
    mMinY = minY;
    mMaxY = maxY;
    //算出结束位置
    mFinalX = startX + (int) Math.round(totalDistance * coeffX);
    // 如果mFinalX 小于最小，或者大于最大，就要调整 mMinX <= mFinalX <= mMaxX
    mFinalX = Math.min(mFinalX, mMaxX);
    mFinalX = Math.max(mFinalX, mMinX);
        
    mFinalY = startY + (int) Math.round(totalDistance * coeffY);
    // 和mFinalY 同理
    mFinalY = Math.min(mFinalY, mMaxY);
    mFinalY = Math.max(mFinalY, mMinY);
}
```
可以看出，`fling`方法主要是赋值当前速度、起始位置、最大最小X和Y的值，和算出来的结束值。跟`startScroll`方法一样，引发滑动还是要靠`invalidate()`方法和`computeScrollOffset()`方法配合。

下面看`computeScrollOffset()`中模式为`fling`时：
```
public boolean computeScrollOffset() {
    if (mFinished) {
        return false;
    }
    //获取当前时间减去开始时间的值，即经过的时间
    int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    //经过的时间小于duration才继续计算
    if (timePassed < mDuration) {
        switch (mMode) {
        case SCROLL_MODE:
            ......
        case FLING_MODE:
            final float t = (float) timePassed / mDuration;//经过时间占总时间的比值
            final int index = (int) (NB_SAMPLES * t);//比值乘100，并转为整数，也就是0到100的整数
            float distanceCoef = 1.f;
            float velocityCoef = 0.f;
            if (index < NB_SAMPLES) {//在100以内，即未完成
                final float t_inf = (float) index / NB_SAMPLES;
                final float t_sup = (float) (index + 1) / NB_SAMPLES;
                final float d_inf = SPLINE_POSITION[index];
                final float d_sup = SPLINE_POSITION[index + 1];
                velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
                distanceCoef = d_inf + (t - t_inf) * velocityCoef;
            }

            mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
            //算出当前X
            mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
            // 如果超出范围，调整到范围以内 mMinX <= mCurrX <= mMaxX
            mCurrX = Math.min(mCurrX, mMaxX);
            mCurrX = Math.max(mCurrX, mMinX);
            //算出当前Y
            mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
            // 如果超出范围，调整到范围以内 mMinY <= mCurrY <= mMaxY
            mCurrY = Math.min(mCurrY, mMaxY);
            mCurrY = Math.max(mCurrY, mMinY);
            //如果等于结束值，就把结束标志设为true
            if (mCurrX == mFinalX && mCurrY == mFinalY) {
                mFinished = true;
            }

            break;
        }
    }
    else {//经过的时间大于或等于duration，就结束计算
        mCurrX = mFinalX;
        mCurrY = mFinalY;
        mFinished = true;
    }
    return true;
}
```