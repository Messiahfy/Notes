#### 1.构造函数
&emsp;&emsp;在构造函数中，初始化和拿到自定义的属性值，使用完`typedArray`后需要`recycle`回收
&emsp;&emsp;第一个构造函数用于在代码中`new`，第二个用于`xml`加载。第三和第四个不会自动调用，需要自己在第一或第二个中来调用。

&emsp;&emsp;`View`除了第一种构造函数没有传入`AttributeSet`，其他都有，一般第一种用于代码构造`view`,其他是`xml`文件生成`view`时调用的，而传入的`AttributeSet`就是由`xml`中设定的所有属性来构造的对象。
**注意**：属性的优先级：`xml` > `style`（直接在`view xml`属性中引用的`style`） > `defStyleAttr` > `defStyleRes` > `theme`
&emsp;&emsp;调用第三个构造函数时，传入参数`defStyleAttr`为当前应用的`theme`中的某个引用另一个`style`的属性`R.attr.xxx`。
&emsp;&emsp;调用第四个构造函数时，传入参数`defStyleRes`为一个`style（R.style.xxx）`,比`defStyleAttr`设置的属性的优先级低，仅用于`defStyleAttr`为`0`或者不能找到时才生效。
<br>
<br>
> 下面的测量、布局、绘制三大流程可参考`DecorView`-->`FrameLayout`-->`View`来分析，`DecorView`是最终的根布局。

#### 2.Measure测量
###### 2.1 MeasureSpec
* `MeasureSpec.EXACTLY` 对应`match_parent`或`具体数值`
* `MeasureSpec.UNSPECIFIED` 一般不用
* `MeasureSpec.AT_MOST` 对应`wrap_content`

最初的`widthMeasureSpec`和`heightMeasureSpec`是在ViewRootImpl中的performTraversals()-->measureHierarchy-->getRootMeasureSpec中生成

&emsp;&emsp;根据`widthMeasureSpec`和`heightMeasureSpec`，自己设定，最后调用`setMeasuredDimension`，确定测量的宽高
&emsp;&emsp;`ViewGroup`的情况需要测量子`view`,   `view`类型只管自己。如果是`ViewGroup`，`onMeasure`中需先调用`measureChildren`或`measureChild`或`measureChildWithMargins`测量子`view`，引发调用子`view`的`measure`、`onMeasure`

> 参考`ViewGroup`的`measureChildWithMargins`方法及`getChildMeasureSpec`方法，可以看出`parent`传给`child`的`onMeasure()`的`MeasureSpec`的规则

&emsp;&emsp;每个`ViewGroup`都有`LayoutParams`内部类，`xml`布局文件解析时会根据子`view`设置的`layout_width`等属性为子`view`设置`LayoutParams`，`ViewGroup`根据子`Veiw`的`LayoutParams`决定调用子`View`的`Measure`和`onMeasure`的`widthMeasureSpec`和`heightMeasureSpec`。
&emsp;&emsp;`onMeasure`中的两个参数都是由父视图传来，可由父视图根据子视图的`layoutParams`中的`lp.width`、`lp.marginLef`t等信息生成（可参照`FrameLayout`的`onMeasure`方法，其中调用父类`ViewGroup`的`measureChildWithMargins`，`measureChildWithMargins`中调用`getChildMeasureSpec`，此中设置了`MeasureSpec`的`mode`和`size`，并传给子视图）

#### 3.Layout 布局
* onLayout
              Layout的作用是ViewGroup用来确定子元素位置，当ViewGroup的位置被确定后，它在onLayout中会遍历所有的子元素并调用其layout方法，在layout方法中onLayout又会被调用。
              一般只有viewGroup子类才需要重写，其中对每个子view都要调用它的layout，设定它相对于本viewGroup的位置
              传入layout和onLayout的都是相对于父布局的左、上、右、下位置

#### 4.Draw 绘制
* onDraw
             一般只有自定义View才重写，用于绘图


#### 5.引发重新布局和绘制
requestLayout()：调用onMeasure()和onLayout()。会调用rootViewImpl的requestLayout。（如果当前View在请求布局的时候，View树正在进行布局流程的话，该请求会延迟到布局流程完成后或者绘制流程完成且下一次布局发现的时候再执行）

invalidate()：只调用本view的onDraw()。当子View调用了invalidate()方法后，会为该View添加一个标记位，同时不断向父容器请求刷新，父容器通过计算得出自身需要重绘的区域，直到传递到ViewRootImpl中，最终触发performTraversals方法，进行开始View树重绘流程(只绘制需要重绘的视图)。
       