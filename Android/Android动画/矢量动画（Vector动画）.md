## 1.概述
&emsp;&emsp;`Vector Drawable`是将`SVG`格式图片转化为在`Android`开发中的`XML`应用形式，在图片缩放时不会出现像素失真。`Vector Drawable`不仅可用来显示静态图片，在`Android 5.0`以上还可以使用`AnimatedVectorDrawable` 类引用 `ObjectAnimator` 和 `AnimatorSet` 来促使 `VectorDrawable` 属性渐变，从而生成动画效果。（`AnimatedVectorDrawable`引用了属性动画类和`VectorDrawable`）

## 2.使用步骤
> `AnimatedVectorDrawable`可以对`VectorDrawable`中的`<group>`和`<path>`执行动画，`<group>`定义`path`的集合或者子`group`，`<path>`元素定义绘制的路径。
#### 2.1 定义`vector drawable`
&#160; &#160; &#160; &#160;想对`VectorDrawable`执行动画，就要对`<path>`或者`<group>`指定`android:name`属性，用于在`AnimatedVectorDrawable`中引用。
`vecor_play.xml`：
```
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="100dp"
    android:height="100dp"
    android:viewportWidth="10"
    android:viewportHeight="10">

    <group
        android:name="playgroup"
        android:pivotX="5x"
        android:pivotY="5">

        <path
            android:name="play"
            android:fillColor="#ff000000"
            android:pathData="M 3,2 L 7,5 L7,5 L3,5z M 3,8 L7,5 L7,5 L3,5z" />
    </group>
</vector>
```
#### 2.2 定义动画
旋转动画`animator_rorate`：
```
<?xml version="1.0" encoding="utf-8"?>
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000"
    android:propertyName="rotation"
    android:valueFrom="0"
    android:valueTo="-90"
    android:valueType="floatType" />
```
路径形状变换动画`animator_paly_pause`：
```
<?xml version="1.0" encoding="utf-8"?>
<objectAnimator xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500"
    android:propertyName="pathData"
    android:valueFrom="M 3,2 L 7,5 L7,5 L3,5z M 3,8 L7,5 L7,5 L3,5z"
    android:valueTo="M 2,2 L 8,2 L8,4 L2,4z M 2,8 L8,8 L8,6 L2,6z"
    android:valueType="pathType" />
```
#### 2.3 定义`AnimatedVectorDrawable`
在上面创建了`vector`和`动画`后，就要用`AnimatedVectorDrawable`把它们联系在一起。
```
<?xml version="1.0" encoding="utf-8"?>
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/vector_play">
    <!-- 此处先引用了vector -->

    <!-- 变化内容，为vector中的path指定动画 -->
    <target
        android:animation="@animator/animator_play_pause"
        android:name="play"/>

    <!-- 旋转，为vector中的group指定动画 -->
    <target
        android:animation="@animator/animator_rotate"
        android:name="playgroup"/>

</animated-vector>
```
#### 2.4 应用在View中
```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/scene_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <Button
        android:id="@+id/button1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="vetor动画"
        android:transitionName="button" />

    <ImageView
        android:id="@+id/image"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:src="@drawable/animated_play_pause" />
    <!-- src引用了AnimatedVectorDrawable -->
</LinearLayout>
```
```
button1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Drawable drawable = imageView.getDrawable();
                if (drawable instanceof Animatable) {
                    ((Animatable) drawable).start();
                }
            }
        });
```
效果如下：
![效果.gif](https://upload-images.jianshu.io/upload_images/3468445-aed6f4b24daeaaa0.gif?imageMogr2/auto-orient/strip)
##3.其他注意点
`VectorDrawable`可以实现的一些效果：
![各种vector动画](https://upload-images.jianshu.io/upload_images/3468445-9393cafa133c1cdc.gif?imageMogr2/auto-orient/strip)

如果想实现点击暂停，再点击又播放的效果，这类多个状态的效果，参考[视图状态动画](https://www.jianshu.com/writer#/notebooks/29639074/notes/34274922)

一些和矢量动画相关的github库：[vectalign](https://github.com/bonnyfone/vectalign)和[RichPath](https://github.com/tarek360/RichPath)