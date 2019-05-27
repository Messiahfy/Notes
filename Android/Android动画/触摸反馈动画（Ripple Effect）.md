## 1.概述
触摸反馈动画，即`API 21`（Android 5.0）开始出现的水波纹效果，类名为`RippleDrawable`，继承关系如下图：
![RippleDrawable继承关系图](https://upload-images.jianshu.io/upload_images/3468445-65287d6e6957fb4c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 2.系统自带的两个水波纹效果
//有边界
```
android:background="?android:attr/selectableItemBackground"
```
在API 21及以上效果如下：
![有边界21及以上.gif](https://upload-images.jianshu.io/upload_images/3468445-f215f9edaf460031.gif?imageMogr2/auto-orient/strip)
在API 21以下效果如下，只是普通的按压时切换颜色：
![有边界21以下.gif](https://upload-images.jianshu.io/upload_images/3468445-d5c835f40d59bcb0.gif?imageMogr2/auto-orient/strip)

//无边界 （要求API 21以上，21以下不可用），范围为view的矩形外接圆
```
android:background="?android:attr/selectableItemBackgroundBorderless" />
```
![无边界的ripple效果](https://upload-images.jianshu.io/upload_images/3468445-d52f26fcacccc8bb.gif?imageMogr2/auto-orient/strip)
## 3.自定义Ripple
&#160; &#160; &#160; &#160;自定义`Ripple`可以设定想要的颜色和形状，`Ripple`在res/drawable中定义。

-------------------------------------
* 情况一：没有内部item，只有水波纹（<ripple>标签就是定义水波纹的），这时没有底部背景，也就是无边界的。
```
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="@color/colorAccent">
</ripple>
```
![自定义水波纹颜色.gif](https://upload-images.jianshu.io/upload_images/3468445-0501b126a759756e.gif?imageMogr2/auto-orient/strip)

* 情况二：设置内部item，则有底部背景，且作为水波纹的边框范围。**item中的drawable还可以设置shape、图片等，水波纹的边框范围也就跟着变化成对应的shape、图片**
```
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="@color/colorAccent">
    <item android:drawable="@color/gray" />
</ripple>
```
![ripple中设置了内部item](https://upload-images.jianshu.io/upload_images/3468445-581566869d78292c.gif?imageMogr2/auto-orient/strip)

* 情况三：设置内部item，且对item设置其id为`@android:id/mask`，这时item只有在点击和按压时才作为形状边框范围，和水波纹一起出现，且不会显示item的内容。
```
<ripple xmlns:android="http://schemas.android.com/apk/res/android"
    android:color="@color/colorAccent">
    <item
        android:id="@android:id/mask"
        android:drawable="@color/gray" />
</ripple>
```
![item设置id mask.gif](https://upload-images.jianshu.io/upload_images/3468445-f20cd74609fde967.gif?imageMogr2/auto-orient/strip)


