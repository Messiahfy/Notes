官方提供了Material风格的常用控件库，依赖坐标为com.google.android.material:material。
## BottomSheet
类似于底部dialog，但通过CoordinatorLayout，支持了更好的交互。

以一个例子来了解BottomSheet的使用。BottomSheet的核心是BottomSheetBehavior，这里我们称呼配置了BottomSheetBehavior的布局为bottomSheet布局。因为用到Behavior，所以bottomSheet布局需要配合CoordinatorLayout来使用，BottomSheet布局将放在CoordinatorLayout内，布局如下：
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <Button
        android:id="@+id/show_bottom_sheet"
        <!-- 省略其他布局属性 -->
         />

    <androidx.coordinatorlayout.widget.CoordinatorLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent">

        <!-- 任意View都可以，例如LinearLayout，内部嵌套子View，这里就简单写了一个View -->
        <View
            android:id="@+id/bottom_sheet"
            android:layout_width="match_parent"
            android:layout_height="600dp"
            android:background="#00ff00"
            android:orientation="vertical"
            app:layout_behavior="@string/bottom_sheet_behavior">
        </View>
    </androidx.coordinatorlayout.widget.CoordinatorLayout>
</androidx.constraintlayout.widget.ConstraintLayout>
```
代码如下：
```
val button: Button = findViewById(R.id.show_bottom_sheet) //按钮，点击控制bottomSheet状态
val bottomSheet: View = findViewById(R.id.bottom_sheet)  //bottomSheet布局
val behavior = BottomSheetBehavior.from(bottomSheet)     //BottomSheetBehavior用于控制bottomSheet行为

button.setOnClickListener {
    //点击按钮，切换bottomSheet布局的展开和折叠状态。还有其他状态可以设置，比如隐藏。我们这里以常用的展开和折叠为例。
    if (behavior.state == BottomSheetBehavior.STATE_EXPANDED) {
        behavior.setState(BottomSheetBehavior.STATE_COLLAPSED);
    } else {
        behavior.setState(BottomSheetBehavior.STATE_EXPANDED);
    }
}
```
点击按钮，或者在bottomSheet布局范围内上下拖动，正常情况可以看到bottomSheet布局在两种高度之间切换，对应展开和折叠两种状态。这里说正常情况可以看到，是因为如果bottomSheet布局较低，低于默认的折叠高度，则不能看到展开和折叠的效果。如果没有看到效果，可以把bottomSheet布局的高度增大，比如match_parent。

那么默认的折叠高度是什么呢？我们先了解一个BottomSheetBehavior的方法`setPeekHeight`，通过它可以设置折叠状态的高度，但必须比原本高度小。在前面，我们并没有使用这个方法来设置折叠状态的高度，所以会使用默认的高度。这个默认的高度，在`setPeekHeight`方法的注释中说的是按照16:9的比例来设置，有点含糊其辞，具体的计算方式可以在BottomSheetBehavior的`calculatePeekHeight`方法中查看。在我自己的常规情况下（竖屏，且CoordinatorLayout铺满屏幕），这个默认高度在大概是bottomSheet的父布局CoordinatorLayout的高度的2/3。

### 常用配置
以下方法都是BottomSheetBehavior类的。
* setPeekHeight：前面已经描述了设置折叠状态高度的作用。
* setHideable：配置bottomSheet布局是否可以进入隐藏状态，默认为false。和折叠、展开一样，可以通过触摸滑动或者代码设置来切换。
* setSkipCollapsed：是否在bottomSheet布局展开一次之后，每次折叠就会跳过折叠状态，而进入隐藏状态，默认为false。
* setDraggable：是否支持触摸滑动，默认为true。
* setFitToContents：展开状态的高度（顶部位置），是否受bottomSheet布局高度限制，默认为true，也就是展开状态的高度最多为bottomSheet布局的高度。如果设置为false，会使bottomSheet布局的顶部位置在展开情况下，不受限制的和CoordinatorLayout顶部对齐。
* setHalfExpandedRatio：设置setFitToContents为false的时候，存在一个半展开状态，这个方法可以设置半展开状态的位置比例，默认为0.5。
* addBottomSheetCallback：设置状态回调。

### 常见场景
* 只有展开和隐藏两种状态：在bottomSheet布局高度较小时（小于前面说的默认折叠高度），折叠和展开是一个高度，所以我们在展开（或折叠）和隐藏状态直接切换就可以了，记得调用`setHideable(true)`。在bottomSheet布局高度较大时，还会存在折叠状态，这时我们可以调用`setPeekHeight(0)`，让折叠状态的高度为0即可。
* 只有折叠和展开两种状态：首先确保`setHideable(false)`，然后要让bottomSheet布局的高度大于折叠状态的高度。比如bottomSheet布局的高度大于默认折叠高度，那么这种情况自然存在折叠和展开两种状态，但也可以通过`setPeekHeight`来调整折叠状态的高度；如果bottomSheet布局的高度小于默认折叠高度，那么就更需要通过`setPeekHeight`来设置一个比bottomSheet布局高度更小的值。
* 其他情况，基本都是以上情况的变种。对于setFitToContents方法，可能有一些特殊场景可以使用。

### BottomSheetDialog、BottomSheetDialogFragment
BottomSheetDialog就是内部布局已经提供了像我们上面写的CoordinatorLayout和bottomSheet布局，bottomSheet布局为一个FrameLayout，我们再设置的view都是在这个FrameLayout中。需要注意的是，这个FrameLayout的高度为wrap_content，所以我们设置的view要注意一下高度，比如设置一个没有child的LinearLayout，高度为match_parent，那么根据布局测量逻辑，LinearLayout的高度会是0，也就看不到任何效果了。其他注意的是，BottomSheetDialog内已经配置了一些功能，比如`setHideable(true)`。

BottomSheetDialogFragment就是内部使用BottomSheetDialog的Fragment。

### 使用记录
* 在底部有可点击控件的情况（比如tab控件），bottomSheet在tab控件的z轴下方，想在tab控件往上滑动把bottomSheet滑动出来，需要在bottomSheet的控件内部存在`isNestedScrollingEnabled()`为true的才能随便滑，因为BottomSheetBehavior内拦截事件的条件有考虑这个因素。但是如果当前触摸范围内的其他层级有可处理滑动事件的控件，比如tab控件和bottomSheet的z轴下方有一个铺满的列表，则BottomSheetBehavior可能不会拦截，此时可以尝试bottomSheet内使用NestedScrollView。具体原因可调试BottomSheetBehavior内拦截事件的各个条件。