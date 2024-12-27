##1.Canvas常用API
| 操作类型 | 相关方法 | 描述 |
|--|--|--|
| 绘制颜色 | drawColor、drawRGB 、drawARGB| 单一颜色画满整个画布 |
| 绘制基本形状 | drawPoint、drawPoints、drawLine、drawLines、drawRect、drawRoundRect、drawOval、drawCircle、drawArc| 依次为点、线、矩形、椭圆、圆、圆弧 | 
| 绘制图片 | drawBitmap、drawPicture| 绘制位图和快照 | 
| 绘制文字 | drawText、drawTextOnPath | 绘制文字、根据路径绘制文字 | 
| 绘制路径 | drawPath | 绘制任意设定的路径，贝塞尔曲线也会用到 |
| 顶点操作 | drawVertices、drawBitmapMesh | 通过对顶点操作可以使图像形变，drawVertices直接对画布作用、 drawBitmapMesh只对绘制的Bitmap作用 |
| 画布剪裁 | clipPath、clipRect、clipOutPath 、clipOutRect  | 设置画布的显示区域，后两者是API 26加入 |
| 画布快照 | save、restore、saveLayerXxx、restoreToCount、getSaveCount | 依次为 保存当前状态、回滚到上一次保存的状态、保存图层状态、回滚到指定状态、获取保存次数 |
| 画布变换 |translate、scale、rotate、skew| 依次为 位移、缩放、 旋转、错切，**实质都是矩阵变换** |
| 矩阵 |getMatrix、setMatrix、concat| 实际上画布的位移或缩放等操作的都是图像矩阵Matrix，不过Matrix难以理解和使用，故封装了一些常用的方法。 |

> 1.`Canvas`中的默认坐标原点是其所在的`View`的左上角坐标，例如一个`ViewGroup`内包含一个`View`，那么`ViewGroup`中调用`onDraw(canvas)`的`canvas`坐标原点就是此`ViewGroup`的左上角，而`View`中调用`onDraw(canvas)`就是此`View`的左上角，可以通过`translate`方法移动。<br>2.一般使用`Canvas`都是`onDraw`传来的，如果使用`Canvas(bitmap)`的构造函数，那么绘制都是在该`bitmap`上，要显示在`View`上还需要对`onDraw`传来的`Canvas`调用`drawBitmap`并传入该`bitmap`。
#### 1.1 详细介绍某些绘制API使用
###### 1.drawRoundRect 圆角矩形
绘制圆角矩形的方法有两个，第二个`API 21`加入的，意思一样，只是把rect拆成四个坐标。
```
drawRoundRect (RectF rect, float rx, float ry, Paint paint)
drawRoundRect (float left, float top, float right, float bottom, float rx, float ry, Paint paint)
```
关键是`rx`和`ry`这两个参数，圆角矩形的一个圆角就是一个1/4的椭圆，`rx`和`ry`就表示这个1/4椭圆的两个半径，如果相等就变成圆。
![圆角矩形](https://upload-images.jianshu.io/upload_images/3468445-cd63a94eff05beee.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 2.drawOval 椭圆
传入一个矩形，椭圆就是这个矩形的内切图形，矩形是正方形的话，椭圆就变成了圆
```
// 第一种
RectF rectF = new RectF(100,100,800,400);
canvas.drawOval(rectF,mPaint);

// 第二种
canvas.drawOval(100,100,800,400,mPaint);
```

###### 3.drawArc 绘制圆弧
绘制圆弧也有两个方法，同样第二个是API 21添加，只是把rect拆成四个坐标
```
// 第一种
public void drawArc(@NonNull RectF oval, float startAngle, float sweepAngle, boolean useCenter, @NonNull Paint paint){}
    
// 第二种
public void drawArc(float left, float top, float right, float bottom, float startAngle,
            float sweepAngle, boolean useCenter, @NonNull Paint paint) {}
```
后面三个参数：
```
startAngle  // 开始角度
sweepAngle  // 扫过角度
useCenter   // 是否使用中心
```
就是以矩形的内切椭圆的圆心为圆弧中心，根据`startAngle`和`sweepAngle`来得到椭圆的一部分即是一个圆弧，而`useCenter `表示是否使用圆心，如下，前者不使用，后者使用。
![绘制圆弧](https://upload-images.jianshu.io/upload_images/3468445-20de57747ce77e20.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 1.2 剪切
`clip`操作就是对`canvas`进行裁剪，然后在调用`canvas`的绘制方法，最终只有裁剪的区域保留绘制的图像。



#### 1.3 画布操作和图层
`translate`、`scale`等操作实质都输入`matrix`矩阵变换，`matrix`中一起分析。
这里分析`save`和`saveLayer`。
###### 1.save和saveLayer
**save**：将当前矩阵变换（包括`translate`等）和`clip`操作保存到专门的堆栈。后面调用`restore()`可以回到当前状态。

**saveLayer**：和`save`一样，但是会新建一个图层来绘制（在新的缓存区offScreen离屏渲染），从方法名来理解`save layer`就是保存当前图层，再新建一个图层绘制。调用`restore()`后才把离屏缓存绘制到当前`canvas`（相当于前一个图层）上。
也就是说`saveLayer`会创建一个全新透明的`bitmap`，大小与指定保存的区域一致，其后的绘图操作都放在这个`bitmap`上进行。在绘制结束后，会直接盖在前一层的`Bitmap`上显示。

通过canvas.saveLayer（）新建一个layer，新建的layer放置在canvas默认layer的上部，当我们执行了canvas.saveLayer（）之后，我们所有的绘制操作都绘制到了我们新建的layer上，而不是canvas默认的layer。

总的来说，可以想象成第一层是真实显示的屏幕，第二层是正常使用的`canvas`，我们平常的绘制都是在`canvas`上绘制后再盖在屏幕上，而对`canvas`的矩阵变换比如`translate`方法就是把第二层这个`canvas`移动，然后绘制在此`canvas`上的图形相对于屏幕即有了偏移。而`saveLayer`方法则是再新建设定了矩形大小的第三层（甚至第四层、第五层），在新的层上绘制，如果上一层调用了矩阵变换例如平移，那么新建的层的坐标原点是以平移后的画布的左上角为准。

`save`和`saveLayer`都返回一个`int`数值，用于`restoreToCount`时传入。

----------------------------------------------------------

## 2.Paint常用API
reset() 
重置画笔    
#### 1.setStyle()
设置画笔模式
```
Paint.Style.STROKE                //描边
Paint.Style.FILL                  //填充
Paint.Style.FILL_AND_STROKE       //描边加填充
```
将画笔的描边宽度设大一点，从下图可以看出三种模式的区别
![画笔三种模式](https://upload-images.jianshu.io/upload_images/3468445-b2cb5dcc0443fee5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
#### 2.stroke 描边
绘制线条，如直线、矩形、圆等等各种图像的边都属于描边。只要不是填充形状内部，就是描边。
###### 2.1setStrokeWidth
设置描边的宽度

###### 2.2 setStrokeCap
设置描边的两端样式
![](https://upload-images.jianshu.io/upload_images/3468445-76c413859d55cd4f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 2.3 setStrokeJoin
转角的的类型
![](https://upload-images.jianshu.io/upload_images/3468445-c2d8c671b5f6d055.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

###### 2.4 setStrokeMiter
![setStrokeMiter](https://upload-images.jianshu.io/upload_images/3468445-2ad8e6d19d17a7c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.设置绘制内容（颜色和图像）
### 3.1 setARGB
### 3.2 setAlpha 
### 3.3 setColor 
### 3.4 setColorFilter
在`setColor`前调用`setColorFilter`，会对`setColor`传入的颜色进行过滤转换，偏移转换成另一种颜色，转换规则根据传入的值而定。
### 3.5 setShader(Shader shader)
对绘制图形除了用颜色填充或描边外，还可以用位图或者渐变来填充（描边好像有点奇葩）。`Shader`渲染`Android`提供了**5**个子类，有`BitmapShader`，`ComposeShader`，`LinearGradient`，`RadialGradient`，`SweepGradient`。`Shader`中有一个`TileMode`，共有**3**种模式：
`CLAMP`：当图片小于绘制尺寸时要进行边界拉伸来填充
`REPEAT`：当图片小于绘制尺寸时重复平铺
`MIRROR`：当图片小于绘制尺寸时镜像平铺

> 用`Canvas`结合`Paint`设置`BitmapShader`来填充圆角矩形，可以实现圆角图片

### 3.6 setMaskFilter(MaskFilter maskfilter)
`MaskFilter`有两个子类，一个`BlurMaskFilter`一个是`EmbossMaskFilter`
下面是`BlurMaskFilter`的效果
![BlurMaskFilter效果](https://upload-images.jianshu.io/upload_images/3468445-421b24f4d0415657.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

而`EmbossMaskFilter`是实现对图像的浮雕效果，看起来像立体雕像。

### 3.7 setShadowLayer(float radius, float dx, float dy, int shadowColor)
绘制阴影如下：
![setShadowLayer效果](https://upload-images.jianshu.io/upload_images/3468445-40259a2d6ce0c8b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.8 setDither(boolean dither)
在内存中用`ARGB888`颜色模型表示图像，在该图像拷贝到屏幕帧缓冲区的过程中，它会变成`RGB565`颜色模式。`RGB565`最多只能表示2^16 =65536种图像，这对于`RGB888`所能表示的2^24 =16777216种颜色来说显然在表现力上要略逊一筹。这集中表现在显示某些带有渐变效果的图片时，出现了一条条的颜色带，而不是原始的平滑的渐变效果。使用抖动可以让效果更好。左边抖动，右边未抖动：
![setDither](https://upload-images.jianshu.io/upload_images/3468445-0f5b18f752b7fac4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.9 setColorFilter(ColorFilter filter)
可以对绘制的`bitmap`的颜色进行转换（实现滤镜效果），应该对纯色而不是位图的绘制也可以转换，参考[Paint之ColorMatrix与滤镜效果](https://blog.csdn.net/harvic880925/article/details/51187277)和[Paint之setColorFilter](https://blog.csdn.net/harvic880925/article/details/51253944)

### 3.10 setXfermode(Xfermode xfermode)
Xfermode就是Transfer mode，可以实现图形混合，参考[Paint之setXfermode](https://blog.csdn.net/harvic880925/article/details/51264653)

PorterDuffXfermode

`setFilterBitmap(boolean filter)` 网上说对位图滤波处理，可以抗锯齿
`setAntiAlias(boolean aa) ` 设置画笔是否抗锯齿 
`clearShadowLayer()` 清除阴影

> 可以参考扔物线的自定义view文章 [Paint 详解](https://rengwuxian.com/ui-1-2/)

## 4.setPathEffect(PathEffect effect)
用于对`canvas.drawPath(path,paint)`画出的路径进行修饰，比如把折线的折角变为曲线，或者实线变虚线，**CornerPathEffect三角形等的角变圆角**等等。
参考[setPathEffec介绍](https://blog.csdn.net/harvic880925/article/details/51010839)

## 5.字体相关
`setTextSize(float textSize) `  设置文字大小 

`setFakeBoldText(boolean fakeBoldText) ` 设置是否为粗体文字 

`setStrikeThruText(boolean strikeThruText) ` 设置带有删除线效果 

`setUnderlineText(boolean underlineText) ` 设置下划线 

`setTextAlign(Paint.Align align) ` 设置开始绘图点位置
Paint.Align.LEFT  drawText传入的点作为文字的左端，向右绘制
Paint.Align.CENTER  作为中点
Paint.Align.RIGHT 作为文字的右端，向左绘制
比如该点是View的中点，那么LEFT就会使文字偏右，RIGHT使偏左，CENTER居中

`setTextScaleX(float scaleX) ` 水平拉伸设置 

`setTextSkewX(float skewX)`  设置字体水平倾斜度，普通斜体字是-0.25，可见往右斜 

`setTypeface(Typeface typeface)` 字体样式 。加粗，倾斜，加粗并倾斜，正常，还可以从资源文件中加载自定义字体。

## 6.测量相关
`measureText` 获得文字的宽度
`getTextBounds` 获得文字的宽高
`getTextSize` 获得文字的尺寸
`getTextWidths` 获得字符串中每个的字符的宽度

https://www.zhihu.com/column/p/20735030