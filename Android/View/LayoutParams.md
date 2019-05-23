如`View`的加载过程中描述，从`xml`中解析创建`view`，`ViewGroup`根据该`view`的`attrs`为其构造`LayoutParams`，将此`LayoutParams`设给`view`。在后面的测量绘制等流程中会把`view`的`LayoutParams`取出来用于测量布局绘制流程。

![LayoutParams常用类继承图](https://upload-images.jianshu.io/upload_images/3468445-ab5c32d6c285bdf3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图是一些最常用的`LayoutParams`类的继承图
所有的`LayoutParams`都继承自`ViewGroup.LayoutParams`，

* `ViewGroup.LayoutParams`：属性只有`width`和`height`，对应`xml`布局文件中的`layout_width`和`layout_height`。
* `ViewGroup.MarginLayoutParams`：除了继承来的宽高，多的有各种方向的`margin`。

* `WindowManager.LayoutParams`：`Activity`、`Dialog`或者自行添加的`Window`中使用。除了继承的宽高属性，还有`gravity`、`x`、`y`，`type`、`flag`**等等**属性。其中比如`x`和`y`用于在设定`gravity`后定义x和y轴的位置偏移；`type`可以用于设定`Window`的级别，在悬浮窗中用处很大。
