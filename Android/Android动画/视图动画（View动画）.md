## 1.视图动画概述
&emsp;&emsp;视图动画位于`android.view.animation`包中，也叫`Tween`（补间）动画，用于对`view`进行`alpha`（透明度），`translate`（位移），`scale`（缩放），`rotate`（旋转）四种动画变化。

继承关系图如下：![视图动画继承关系](https://upload-images.jianshu.io/upload_images/3468445-299039fb85c23c80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

| Java 类名 | xml 关键字 | 描述信息 |
| ------ | ------ | ------ |
| AlphaAnimation | <alpha> 放置在 res/anim/ 目录下| 透明度变化动画效果  |
| RotateAnimation | <rotate> 放置在 res/anim/ 目录下 |旋转动画效果 |
| ScaleAnimation | <scale> 放置在 res/anim/ 目录下 | 尺寸缩放动画效果 |
| TranslateAnimation | <translate> 放置在 res/anim/ 目录下 | 位置移动动画效果 |
| AnimationSet | <set> 放置在 res/anim/ 目录下 | 持有其它动画元素 alpha、scale、translate、rotate 或者嵌套其它 set 元素的组合动画容器 |

&#160; &#160; &#160; &#160;视图动画对`view`的变换，并没有改变`view`的实际位置，`view`的事件响应范围还是原本的位置。
## 2.视图动画的属性
#### 2.1 Animation属性
`AlphaAnimation`、`RotateAnimation`、`ScaleAnimation`、 `TranslateAnimation`、`AnimationSet`都会继承`Animation`的属性
| XML属性 | Java方法 | 描述 |
| ------ | ------ | ------ |
| android:detachWallpaper | setDetachWallpaper(boolean)| 是否在壁纸上运行 |
| android:duration| setDuration(long)| 动画运行时长，毫秒 |
| android:fillAfter | setFillAfter(boolean)| 动画结束时是否保持动画最后的状态 |
| android:fillBefore | setFillBefore(boolean)| 动画结束时是否还原到开始动画前的状态 |
| android:fillEnabled | setFillEnabled(boolean)| 设为true才考虑android:fillBefore |
| android:interpolator | setInterpolator(Interpolator)	| 定义动画移动的插值器 |
|android:repeatCount | setRepeatCount(int)| 动画重复次数 |
|android:repeatMode | setRepeatMode(int)| 重复模式有两个值，reverse表示倒序回放，restart表示从头播放 |
|android:startOffset | setStartOffset(long)| 调用start函数之后等待开始运行的时间，毫秒 |
|android:zAdjustment | setZAdjustment(int)| 设置动画的内容运行时在Z轴上的位置（top/bottom/normal），默认为normal，暂不知具体效果 |
&#160; &#160; &#160; &#160;以上属性是任何一种视图动画都具有的属性，`AnimationSet`设置某个属性就会覆盖它的所有子动画的对应属性。
#### 2.2 Alpha属性
| XML属性 | Java方法 | 描述 |
| ------ | ------ | ------ |
| android:fromAlpha| 构造函数| 动画开始的透明度（0.0到1.0，0.0是全透明，1.0是不透明）|
| android:toAlpha| 构造函数| 动画结束的透明度，同上|
#### 2.3 Rotate属性
| XML属性 | Java方法 | 描述 |
| ------ | ------ | ------ |
| android:fromDegrees| 构造函数| 旋转开始角度，正代表顺时针度数，负代表逆时针度数|
| android:toDegrees| 构造函数| 旋转结束角度，同上|
| android:pivotX| 构造函数| 旋转中心的X坐标，（数值，百分数，百分数p），如50表示当前view的left坐标加上50px，50%表示当前view的left坐标加上自身宽度的50%，50%p表示当前view的left坐标加上父容器宽度的50%|
| android:pivotY| 构造函数| 旋转中心的Y坐标，规则同上|
#### 2.4 Scale属性
| XML属性 | Java方法 | 描述 |
| ------ | ------ | ------ |
| android:fromXScale| 构造函数| 初始X轴缩放比例，原本宽高的比值，默认为0|
| android:toXScale| 构造函数| 结束X轴缩放比例，同上|
| android:fromYScale| 构造函数| 初始Y轴缩放比例，同上|
| android:toYScale| 构造函数| 结束Y轴缩放比例，同上|
| android:pivotX| 构造函数|缩放中心的X坐标，（数值，百分数，百分数p），规则同Rotate|
| android:pivotY| 构造函数| 缩放中心的Y坐标，规则同上|
#### 2.5 Translate属性
| XML属性 | Java方法 | 描述 |
| ------ | ------ | ------ |
| android:fromXDelta| 构造函数| 初始X轴位置，（数值，百分数，百分数p）|
| android:fromYDelta | 构造函数| 初始Y轴位置，同上|
| android:toXDelta | 构造函数| 结束X轴位置，同上|
| android:toYDelta | 构造函数| 结束Y轴位置，同上|
#### 2.6 AnimationSet
| XML属性 | Java方法 | 描述 |
| ------ | ------ | ------ |
| android:shareInterpolator | 构造函数| 是否用AnimationSet的差值器覆盖子动画的插值器，默认为true|
## 3.代码构造视图动画
&#160; &#160; &#160; &#160;用代码构造视图动画基本就是使用构造函数设置特有属性，然后再对构造的对象调用set...方法设置继承自Animation的属性。如下：
```
TranslateAnimation translateAnimation = new TranslateAnimation(20, 20, 20, 20);
translateAnimation.setDuration(2000);
translateAnimation.setFillAfter(true);
```
&#160; &#160; &#160; &#160;在代码中构造视图动画，还有可以设置数值类型的构造函数，如下：
```
TranslateAnimation translateAnimation = new TranslateAnimation(Animation.RELATIVE_TO_SELF, 20, Animation.RELATIVE_TO_SELF, 20, Animation.RELATIVE_TO_SELF, 20, Animation.RELATIVE_TO_SELF, 20);
```
&#160; &#160; &#160; &#160;这样就相当于`xml`中设置`fromXDelta`等值为20%，这些Animation中的**常用**静态变量如下：
| 静态变量 | 值 | 描述 |
| ------ | ------ | ------ |
| INFINITE | -1 | 用于设置repeatCount，表示无穷次|
| RESTART| 1 | 用于设置repeatMode，表示从头播放|
| REVERSE| 2 | 用于设置repeatMode，表示倒序回放|
| ABSOLUTE| 0| 用于设置坐标，数值|
| RELATIVE_TO_SELF| 1| 用于设置坐标，百分数|
| RELATIVE_TO_PARENT| 2 | 用于设置坐标，百分数p|
## 4.运行视图动画
#### 4.1运行视图动画的相关方法
```
Animation animation= AnimationUtils.loadAnimation(this, R.anim.xxx);
//或代码创建
//TranslateAnimation animation = new TranslateAnimation(20, 20, 20, 20);
view.startAnimation(animation);
```
某些关于运行动画方面的方法如下：
| Animation 类的方法| 描述 |
| ------ | ------ |
| reset() | 重置 Animation 到初始状态（继续运行） |
| cancel() | 回调AnimationListener 的end（不会停止动画，只是改变一些状态标志位等） |
| start() | 设置关于开始的状态标志 |
| setAnimationListener(AnimationListener listener) | 设置动画监听器 |
| hasStarted() | 检测是否开始 |
| hasEnded() | 检测是否结束 |

| View 类的方法| 描述 |
| ------ | ------ |
| startAnimation(Animation animation) | 开始运行动画 |
 | clearAnimation() | 清除、停止动画 |
#### 4.2 按序执行动画
* `AnimationSet`中多个动画，默认同时执行。要按顺序执行，可以设置`android:startOffset`对某个动画设置开始执行的偏移时间达到顺序效果。
* 也可以代码设置动画监听器，多个视图动画对象，在监听器回调end时再去执行其他动画
## 5.Interpolator（插值器）
&#160; &#160; &#160; &#160;`Interpolator`的继承关系图如下：
![Interpolator继承关系图](https://upload-images.jianshu.io/upload_images/3468445-e93fe18072cf6f73.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
&#160; &#160; &#160; &#160;在视图动画中一般常用的是`BaseInterpolator`的子类。

#### 6.视图动画原理
&#160; &#160; &#160; &#160;`View`的`startAnimation`方法如下，设置了动画，且调用了`invalidate(true)`
```
    public void startAnimation(Animation animation) {
        animation.setStartTime(Animation.START_ON_FIRST_FRAME);
        setAnimation(animation);
        invalidateParentCaches();
        invalidate(true);
    }
```
&#160; &#160; &#160; &#160;而`invalidate(true)`又会走到`ViewRootImpl `的 `scheduleTraversals()`。绘制流程中`ViewGroup`调用`dispatchDraw() `将绘制事件通知给子 `View`。`ViewGroup` 重写了 `dispatchDraw()`，调用了 `drawChild()`，而 `drawChild()` 调用了子 `View` 的 `draw(Canvas, ViewGroup, long)`，而这个方法又会去调用到 `draw(Canvas)`。
&#160; &#160; &#160; &#160;在`draw(Canvas canvas, ViewGroup parent, long drawingTime)`中有以下代码：
```
final Animation a = getAnimation();
if (a != null) {
    more = applyLegacyAnimation(parent, drawingTime, a, scalingRequired);//此方法中..
    concatMatrix = a.willChangeTransformationMatrix();
    if (concatMatrix) {
        mPrivateFlags3 |= PFLAG3_VIEW_IS_ANIMATING_TRANSFORM;
    }
    transformToApply = parent.getChildTransformation();
} 
......
```
可见，在绘制的时候会判断动画不会空，调用`applyLegacyAnimation(...)`方法，其中会调用如下：
```
...
final Transformation t = parent.getChildTransformation();
boolean more = a.getTransformation(drawingTime, t, 1f);//调用animation的getTransformation方法
...
```
animation的getTransformation方法则会调用到`Animation`类的`applyTransformation`方法，此方法是`Animation`的子类都要重写的。  
此方法用于设置传入的`Transformation`的矩阵或者`alpha`。所以视图动画实质都是用来操作`Transfomation`对象中的矩阵和alpha值，

最终得到的`transformToApply`，其中包含矩阵变换和alpha值的信息，后续代码会使用`transformToApply`用于  
`canvas`的矩阵变换和透明度变化。补间动画其实只是调整了子view画布canvas的坐标系，其实并没有修改任何属性，所以只能在原位置才能处理触摸事件

> 设置了视图动画后，就会引发view重绘，绘制时执行矩阵变化。如果动画没有结束，即在View的applyLegacyAnimation方法中判断动画的getTransformation返回为true，就还会调用parent的invalidate，继续下一帧绘制。

```
## 7.视图动画的其他应用
----------------------------------------------
 视图动画除了用在`view`，还可以用在：
* `PopupWindow`中`setAnimationStyle(int)`
* `Activity`的`overridePendingTransition(int,int)`，将此方法跟在startActivity()或finish()后面可以实现页面切换动画；
或者在`activity`中调用`getWindow().setWindowAnimations(R.style.anim)`（`dialog`也可以用这种方式实现动画，因为`dialog`也有`window`），或者在主题中设置如下属性：
```
<!--这种方式就只有activity可以了，因为主题只会应用于`activity`-->
<item name="android:windowAnimationStyle">@style/anim</item>
```
`anim`这个`style`如下：
```
<style name="anim">
       <!--下面两个activity和dialog都有效，因为它们都有window-->
       <item name="android:windowEnterAnimation">@android:anim/slide_in_left</item>
       <item name="android:windowExitAnimation">@android:anim/slide_out_right</item>
        <!--下面就只对activity有效了-->
        <item name="android:activityOpenEnterAnimation">@android:anim/slide_in_left</item>
        <item name="android:activityOpenExitAnimation">@android:anim/slide_out_right</item>
        <item name="android:activityCloseEnterAnimation">@android:anim/slide_in_left</item>
        <item name="android:activityCloseExitAnimation">@android:anim/slide_out_right</item>
    </style>
```
* 也可以用于`Fragment`的切换动画，`FragmentTransaction`可以调用`setCustomAnimations(R.anim.xxx,R.anim.xxx)`方法（也可以使用`animtor`）设置入场退场动画，还有一个重载`setCustomAnimations(enter, exit,  popEnter, popExit)`，后两者动画在`popBackStack()`时作用。
* 给`ViewGroup`设置`android:layoutAnimation="@anim/xxx"`（xml中使用<layoutAnimation/>包装视图动画），或者代码中调用`setLayoutAnimation(LayoutAnimationController controller)`，实现子控件第一次显示的进场动画效果。适用于所有ViewGroup。可以看出这里使用的是经过了一层包装的视图动画。
----------------------------------------------