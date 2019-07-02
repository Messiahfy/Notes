[死磕Android_View工作原理你需要知道的一切](https://blog.csdn.net/xfhy_/article/details/90270630)
## 1.view的加载方式
1. 开发者直接使用构造函数
2. 使用`layoutInflater.inflate`方法加载`xml`（内部通过xml分析节点名，最终通过反射方式创建view对象，还是会调用构造函数）
--------------------------------------------------------------------
## 2.加载过程
在`Window`的分析中已经有了相关的部分，这里具体分析`View`绘制流程。
`Activity` -- `PhoneWindow` -- `DecorView(FrameLayout)`
1. `Activity`的`setContentView`实际调用`PhoneWindow`的`setContentView`
2. `PhoneWindow的setContentView()`方法中调用了`LayoutInflater`的`inflate()`方法来填充布局，创建`DecorView`并作为`root`
3. `LayoutInflater`的`inflate()`方法递归分析完xml标签树，完成所有`view`的创建和设置，此时每个`view`的对象和其`layoutParams`都已创建（`layoutParams`由`LayoutInflater`内部根据`view`的`xml`属性生成的`AttributeSet`中的相关信息而构造）。核心代码如下：
```
//LayoutInflater.rInflate()方法内的部分关键代码
final View view = createViewFromTag(parent, name, context, attrs);  //构造view
final ViewGroup viewGroup = (ViewGroup) parent;  //view的父view
final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);  //根据attrs为view构造LayoutParams
rInflateChildren(parser, view, attrs, true);  //继续加载view 的子view
viewGroup.addView(view, params);  //调用addView把view添加到viewGroup
```
先构造实例，再构造`LayoutParams`，然后递归加载`children`，加载后将自己加入到`parent`布局中。所以自定义View用于XML中时，在构造函数中不要设置自身的`LayoutParams`，因为会被后续覆盖；<br/>
而在代码中创建View时则需自行创建`LayoutParams`，否则父布局会创建默认的；自定义View中构造函数添加的`children`比XML中写的`children`先添加；

> **总结**：`onCreate()`中调用`setContentView()`，会创建好所有`View`对象，但要`onResume()`之后才开始绘制。
## 3.测量、布局、绘制
`View`的绘制（包含`measure`、`layout`、`draw`）是由`ViewRootImpl`来负责的。每个应用程序窗口的`decorView`都有一个与之关联的`ViewRootImpl`对象。
5. 在`Window`的分析中，知道在`onResume`后会调用`WindowManagerImpl.addView` -> `WindowManagerGlobal.addView`->`ViewRootImpl.setView`，在`ViewRootImpl`的`setView`方法中会调用`requestLayout()`，以完成应用程序用户界面的初次布局。
```
//ViewRootImpl.java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();//检查是是否在主线程
        mLayoutRequested = true;//设置标志位，为ture才执行measure和layout
        scheduleTraversals();//向主线程发送一个“遍历”消息，最终会导致ViewRootImpl的performTraversals()方法被调用
    }
}

void scheduleTraversals() {
    if (!mTraversalScheduled) {//同一帧内不会多次调用遍历
        mTraversalScheduled = true;
        //发送一个同步屏障消息，MessageQueue中此消息之后的同步消息都不会被处理
        //而是优先处理异步消息
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //Choreographer发送一个包含mTraversalRunnable的异步消息，
        //mTraversalRunnable的run方法是doTraversal()
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```
&emsp;&emsp;`scheduleTraversals`方法中先发送同步屏障，然后发送包含`mTraversalRunnable`的异步消息，保证此异步消息先执行，即先执行`doTraversal()`方法，保证视图及时刷新。
关于**同步屏障**可以参考[这篇文章](https://blog.csdn.net/asdgbc/article/details/79148180)

&emsp;&emsp;`Choreographer`的`postCallback`方法会把`mTraversalRunnable`放到一个回调队列数组中，然后调用`scheduleFrameLocked`->`scheduleVsyncLocked`，最终调用`Native`方法向底层注册监听下一个屏幕刷新信号`VSync`，当下一个`VSync`发出时，底层就会回调 `Choreographer `的`onVsync()` ；
&emsp;&emsp;`onVsync() `方法被回调时，会往主线程的消息队列中发送一个执行 `doFrame() `方法的消息，这个消息是异步消息，所以不会被同步屏障拦截住；
&emsp;&emsp;`doFrame() `方法会去取出之前放进回调队列数组里的任务`mTraversalRunnable`来执行，取出来的这个任务实际上是` ViewRootImpl `的 `doTraversal()` 操作
可参考[屏幕刷新机制1](https://blog.csdn.net/qian520ao/article/details/80954626)和[屏幕刷新机制2](https://www.cnblogs.com/dasusu/p/8311324.html)
> **总结**：一帧内会过滤`scheduleTraversals()`的多次调用。**同步屏障**在下一个**Vsync**信号到来引发执行`doTraversal()`才会删除，导致同步消息都会等到绘制之后才执行，保证绘制的优先。
```
void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //移除同步屏障消息
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        ...
        
        performTraversals();
        ...
    }
}
```

6. 执行到了`viewRootImpl`中的`performTraversals()`中，大致流程为：
* **measure**： `performMeasure` --> `view.measure`(从`DecorView`开始) --> `view.onMeasure`(`DecorView`父类`FrameLayout`的`onMeasure`)，如果是`ViewGroup`，`onMeasure`中需先调用`measureChildren`或`measureChild`或`measureChildWithMargins`测量子`view`，则会调用子view的`measure`，`onMeasure`。即遍历所有`view`（`ViewGroup`）的`onMeasure`。

**注意**：measureChildWithMargins才会在测量子view的时候考虑margin，而measureChildren和measureChild不支持margin。measureChildWithMargins中用的是MarginLayoutParams，另两者用的是ViewGroup.LayoutParams。使用measureChildWithMargins时，如果用addView且没有设置LayoutParams，会默认设置ViewGroup.LayoutParams，会和measureChildWithMargins冲突，所以需要重写generateDefaultLayoutParams（和generateLayoutParams），构造MarginLayoutParams或其子类。FrameLayout和LinearLayout等都是如此。
           onMeasure的调用由父到子，方法的返回顺序由子到父。父用自身layout_width和layout_height和MeasureSpec以及子的layout_width和layout_height来为子设定MeasureSpec，子得到MeasureSpec完成测量，父再根据子的测量宽高和自身lw，lh和自身MeasureSpec来设定自身宽高。

* **layout**：`performLayout` --> `view.layout`(从`DecorView`开始) --> `view.onLayout`（`DecorView`是`FrameLayout`的`onLayout`，其中调用每个子`view`的`layout`，子`view`的`layout`中会调用`onLayout`）即遍历所有`view`（`ViewGroup`）的`onLayout`

* **draw**：`performDraw` --> `draw`  --> 选择硬件绘制或软件绘制 --> `view.draw `（从`DecorView开始`）-->`view.onDraw`

&emsp;&emsp;`view`的`draw`先绘制`background`，再调用`onDraw`后，还会调用`dispatchDraw`，`dispatchDraw`方法在`view`中为空方法，其在`viewGroup`中实现，调用`drawChild`方法，`drawChild`会调用了每个子`view`的`draw(Canvas canvas, ViewGroup parent, long drawingTime)`继而调用`draw(Canvas canvas)`-->`onDraw`，从而绘制每个子`view `。注意视图树的绘制使用的是一个Canvas，ViewGroup调用child的`draw(Canvas canvas, ViewGroup parent, long drawingTime)`方法会修改Canvas的translate等，使之对应child的坐标。


**绘制流程中注意的是**，`view`的`onMeasure`只管自己，`ViewGroup`还要管子`view`。`view`一般不用重写`onLayout`，`ViewGroup`需要在`onLayout`中调用子`View`的`layout`为其设置布局位置。`view`重写`onDraw`绘制，`ViewGroup`一般不用重写`onDraw`。

> 测量、布局、绘制的具体过程在自定义View中描述