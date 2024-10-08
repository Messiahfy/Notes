## 1.构造函数
&emsp;&emsp;在构造函数中，初始化和拿到自定义的属性值，使用完`typedArray`后需要`recycle`回收
&emsp;&emsp;第一个构造函数用于在代码中`new`，第二个用于`xml`加载。第三和第四个不会自动调用，需要自己在第一或第二个中来调用。

&emsp;&emsp;`View`除了第一种构造函数没有传入`AttributeSet`，其他都有，一般第一种用于代码构造`view`,其他是`xml`文件生成`view`时调用的，而传入的`AttributeSet`就是由`xml`中设定的所有属性来构造的对象。
**注意**：属性的优先级：`xml` > `style`（直接在`view xml`属性中引用的`style`） > `defStyleAttr` > `defStyleRes` > `theme`
&emsp;&emsp;调用第三个构造函数时，传入参数`defStyleAttr`为当前应用的`theme`中的某个引用另一个`style`的属性`R.attr.xxx`。
&emsp;&emsp;调用第四个构造函数时，传入参数`defStyleRes`为一个`style（R.style.xxx）`,比`defStyleAttr`设置的属性的优先级低，仅用于`defStyleAttr`为`0`或者不能找到时才生效。
<br>
<br>
> 下面的测量、布局、绘制三大流程可参考`DecorView`-->`FrameLayout`-->`View`来分析，`DecorView`是最终的根布局。

## 2.Measure测量
### 2.1 MeasureSpec
MeasureSpec是一个32位的int数值，由高2位的SpecMode和低30位的SpecSize两部分组成，通过`MeasureSpec.getMode(int measureSpec)`和`MeasureSpec.getSize(int measureSpec)`可以获得其模式和尺寸。<br/>
模式有三种：
* `MeasureSpec.EXACTLY` 表示父控件希望子控件使用确定的尺寸
* `MeasureSpec.UNSPECIFIED` 表示父控件没有限制子控件的尺寸
* `MeasureSpec.AT_MOST` 表示父控件希望子控件自行确定尺寸，但不能超过父控件

SpecMode和`MATCH_PARENT`、`WRAP_CONTENT`的对应关系可以参考ViewGroup的`getChildMeasureSpec`方法

> 对于`MeasureSpec.UNSPECIFIED`，可以参考ScrollView，因为ScrollView的子view可以无限高，所以测量高度的时候，使用`MeasureSpec.UNSPECIFIED`，不限制高度，由子view自己决定。

### 2.2 MeasureSpec的设定
最初的`widthMeasureSpec`和`heightMeasureSpec`是在ViewRootImpl中的performTraversals()-->measureHierarchy-->getRootMeasureSpec中生成

&emsp;&emsp;**自定义View**时根据`widthMeasureSpec`和`heightMeasureSpec`中的模式和尺寸，以及自定义View的特定情况，在最后调用`setMeasuredDimension`，确定测量的宽高。

&emsp;&emsp;**自定义ViewGroup**时，需要先测量`子view`。在`onMeasure`中要先调用`measureChildren`或`measureChild`或`measureChildWithMargins`测量子`view`，引发调用子`view`的`measure`、`onMeasure`。测量了`子view`后，得到所有`子view`的测量尺寸，再结合自身布局特定和`子view`的测量尺寸来调用`setMeasuredDimension`，确定自身测量的宽高。<br/>

> 在调用`setMeasuredDimension`时，传入的宽高一般是使用`resolveSize`或`resolveSizeAndState`得到的。两者都会传入view自身想要的尺寸和父级传来的MeasureSpec，自身想要的尺寸会结合minWidth、minHeight、maxWidth、maxHeight、padding和自身内容等数据。测量的尺寸是一个32位int值，它的高8位表示测量状态，低24位表示实际的尺寸大小，两个方法有区别的是，`resolveSize`返回的int值高8位均为0，也就是只包含尺寸大小，`resolveSizeAndState`则包含了测量的状态。

**注意**：在测量、布局、绘制前，所有View的LayoutParams都是已经构造好的。

> 参考`ViewGroup`的`measureChildWithMargins`方法及`getChildMeasureSpec`方法，可以看出`parent`传给`child`的`onMeasure()`的`MeasureSpec`的规则

&emsp;&emsp;每个`ViewGroup`都有`LayoutParams`内部类，`xml`布局文件解析时会根据子`view`设置的`layout_width`等属性为子`view`设置`LayoutParams`（也可以代码设定），`ViewGroup`根据子`Veiw`的`LayoutParams`决定传给子`View`的`Measure`和`onMeasure`的`widthMeasureSpec`和`heightMeasureSpec`。<br/>
&emsp;&emsp;`onMeasure`中的两个参数都是由父视图传来，是由父视图根据子视图的`layoutParams`中的`lp.width`、`lp.marginLeft`等信息生成（可参照`FrameLayout`的`onMeasure`方法，其中调用父类`ViewGroup`的`measureChildWithMargins`，`measureChildWithMargins`中调用`getChildMeasureSpec`，此中设置了`MeasureSpec`的`mode`和`size`，并传给子视图）

> MeasureSpec会根据LayoutParams得到，用于测量流程，所以一次测量、布局、绘制流程中，修改LayoutParams不会生效，因为不会再使用它，需要在下次流程中测量之前才会再使用它。

**注意**：测量过程可能会有多次，例如parent为wrap_content，child为match_parent，这种情况LinearLayout就会先测量其他所有非match_parent的child，得到其中child的最大尺寸后，再用这个尺寸作为自身尺寸来测量match_parent的child。

## 3.Layout 布局
每个View都是先被父控件调用了`layout(int l, int t, int r, int b)`方法，其中会先调用`setFrame(int l, int t, int r, int b)`更新mLeft、mTop、mRight、mBottom，并调用`invalidate`。`setFrame`之后调用`onLayout(boolean changed, int l, int t, int r, int b)`，View不用重写，ViewGroup需要重写来确定child的位置。
* onLayout

         Layout的作用是ViewGroup用来确定子元素位置，当ViewGroup的位置被确定后，它在onLayout中会遍历所有的子元素并调用其layout方法，在layout方法中onLayout又会被调用。
         viewGroup子类才需要重写，其中对每个子view都要调用它的layout，设定它相对于本viewGroup的位置
         传入layout和onLayout的都是相对于父布局的左、上、右、下位置，ViewGroup根据自身布局特定及子View的尺寸和margin来设定子view的layout方法的传入位置参数。

## 4.Draw 绘制
draw(canvas)方法会先drawBackground(canvas)，然后调用onDraw(canvas)绘制自身，接着调用dispatchDraw(canvas)绘制children。
绘制child会调用drawChild(Canvas canvas, View child, long drawingTime)，其中继而调用draw(canvas)-->onDraw(canvas)
* onDraw
             一般只有自定义View才重写，用于绘图


## 5.引发重新布局和绘制
requestLayout()：调用onMeasure()和onLayout()。会调用rootViewImpl的requestLayout。（如果当前View在请求布局的时候，View树正在进行布局流程的话，该请求会延迟到布局流程完成后或者绘制流程完成且下一次布局发现的时候再执行）

invalidate()：只调用本view的onDraw()。当子View调用了invalidate()方法后，会为该View添加一个标记位，同时不断向父容器请求刷新，父容器通过计算得出自身需要重绘的区域，直到传递到ViewRootImpl中，最终触发performTraversals方法，进行开始View树重绘流程(只绘制需要重绘的视图)。

* 关闭硬件加速则从 DecorView 开始往下的所有子 View 都会被重新绘制。
* 开启硬件加速则只有调用 invalidate 方法的 View 才会重新绘制。

自定义view
https://ke.qq.com/course/313640?taid=2323607372220712
https://ke.qq.com/course/314344?taid=2415429478042600
https://ke.qq.com/course/314351?taid=2363241330428911