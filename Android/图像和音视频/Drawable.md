## 1.概述
&emsp;&emsp;`Drawable`表示可绘制的对象，没有事件交互，仅仅考虑绘制。可以是一个位图（`BitmapDrawable`）或者图形（`ShapeDrawable`）等。
&emsp;&emsp;`Drawable`绘制图形实际也是在自身的`draw(canvas)`方法中调用**Canvas**来绘制图像。
总结：drawable就是在指定范围使用canvas绘制图像

## 2.自定义
&emsp;&emsp;一般使用系统自带的`BitmapDrawable`、`ShapeDrawable`、`LayerDrawable`、`ColorDrawable`等类型，但我们还可以自定义`Drawable`。

draw方法中使用canvas默认坐标原点是drawable

如果对drawable调用了setBounds方法，而draw方法中又通过getBounds方法获取坐标来绘制，那么setBounds可以产生影响。但如果draw方法中并没有使用bounds相关坐标，而是自行设定，那么setBounds并没有用


由于各种drawable的某些属性可以在代码中动态修改，所以使用drawable也可以实现一些动态变化效果。

TransitionDrawable
ScaleDrawable
ClipDrawable