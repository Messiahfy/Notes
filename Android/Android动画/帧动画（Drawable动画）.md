## 1.帧动画概述
帧动画就是`Drawable`动画，本质就是一种`Drawable`，实现像幻灯片一样的效果。对应的类是`AnimationDrawable`，位于`android.graphics.drawable`包中。
![AnimationDrawable继承关系.jpg](https://upload-images.jianshu.io/upload_images/3468445-4010237e059e13e7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 2.定义
定义方式有xml和代码两种
#### 2.1 xml中定义
放置在res/drawable/目录下
```
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="false"    //true表示只运行一次动画，false表示重复运行
    android:visible="true"     //官网说是设置初始的可见状态，默认为false。但测试无效，都会显示，某些文章说xml中并没有解析这个属性。
    android:variablePadding="true">  //表示是否支持可变的Padding。false表示固定使用所有帧中最大的Padding，true表示使用当前帧的padding。默认false
    <item
        android:drawable="@drawable/XXXXX"    //引用的 drawable
        android:duration="150" />             //此item的显示时长
    ......多个item
</animation-list>
```
#### 2.2 代码中定义
```
AnimationDrawable  animationDrawable= new AnimationDrawable();
animationDrawable.addFrame(getResources().getDrawable(R.XXX), 200);
animationDrawable.addFrame(getResources().getDrawable(R.XXX), 200);
......添加任意个帧
animationDrawable.setOneShot(false);
```
## 3.对View设置background
* 如果是`xml`中定义的
可以直接在`xml`设置`background`属性，引用该`xml`中定义的 `animation drawable`
```
<View
        android:layout_width="100dp"
        android:layout_height="100dp"
        app:background="@drawable/animation_drawable />
```
也可以在代码中设置background，引用R.XXX
```
//或者代码中设置
view.setBackground(getResources().getDrawable(R.XXX));
//或者
view.setBackgroundResource(R.XXX);
```
* 如果是在代码中定义的，就只能在代码中设置`background`
```
view.setBackground(animationDrawable);
```
## 4.开始动画
调用`AnimationDrawable`的`start()`方法开始帧动画
```
//如果代码中定义的可以直接用，但还是要把它设为某个view的background后才行。
//AnimationDrawable  animationDrawable= new AnimationDrawable();
//animationDrawable.start();

rocketAnimation = rocketImage.getBackground();
//AnimationDrawable实现了Animatable接口，平时写一般强制转为AnimationDrawable类型
if (rocketAnimation instanceof Animatable) {
    ((Animatable)rocketAnimation).start();
}
```
调用`AnimationDrawable`的`stop()`方法停止帧动画
```
animationDrawable.stop();
```
**注意：**官网说不要在`Activity.onCreate(Bundle)`中调用`start()`方法，因为`AnimationDrawable`尚未完全附加到窗口。 如果想立即播放动画而不需要交互，那么您可能希望从活动中的`Activity.onWindowFocusChanged（boolean）`方法调用它，当Android将窗口置于焦点时将调用该方法。（但自己测试在onCreate()调用start()也正常播放动画了，暂不清楚具体原因）