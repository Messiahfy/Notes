## 概述
谷歌提供了很多Material风格控件，例如AppBarLayout、FloatingActionButton等控件，这些控件都具有控件之间联动的交互效果。它们都需要配合CoordinatorLayout，作为CoordinatorLayout的child来使用。CoordinatorLayout自身是一个ViewGroup，并且是NestedScrollingParent3的子类。

## Behavior方法介绍
CoordinatorLayout作为事件的中介，它要把各种事件发送给需要监听事件的各个子view，那么就需要判定子View是否需要接收事件，CoordinatorLayout的做法就是自定义了LayoutParams，LayoutParams中包含了Behavior字段，只要子View需要接受事件，就给自己设置一个Behavior。Behavior类中有各种方法，对应处理各种事件，CoordinatorLayout中的直接子View可以设置一个自定义的Behavior实例，在自定义Behavior类中编写对各种事件的处理方式。同时，我们也可以学习这种代码思想，将处理事件的行为抽离到Behavior中，让View和对事件的处理解耦，Behavior可以用到任何View中，方便复用。

我们先来看看Behavior类中有哪些方法，看Behavior中的方法时，可以配合到Coordinate中去看调用的地方的源码，以此了解整个流程
```
public static abstract class Behavior<V extends View> {

    public Behavior() {
    }
    
    // 布局文件中加载Behavior时，使用此构造函数，attrs就是使用Behavior的view的attrs，可以根据attrs的属性，做一些初始化操作。源码可见CoordinatorLayout的parseBehavior方法
    // 可以自定义一些属性，用在Behavior关联的view上，Behavior可以在这里获取到
    public Behavior(Context context, AttributeSet attrs) {

    }

    // 在构造view的CoordinatorLayout.LayoutParams，或者代码set时，会调用此方法，
    public void onAttachedToLayoutParams(@NonNull CoordinatorLayout.LayoutParams params) {

    }

    // 设置Behavior时，如果此前有设置其他实例，那么此前的实例会被调用
    public void onDetachedFromLayoutParams() {

    }

    // CoordinatorLayout的onInterceptTouchEvent中，会遍历children的Behavior的onInterceptTouchEvent方法。CoordinatorLayout的源码可见，只要有一个Behavior返回了true，就不会再调用其他Behavior的onInterceptTouchEvent
    public boolean onInterceptTouchEvent(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull MotionEvent ev) {
        return false;
    }

    // 拦截了事件的Behavior会被调用onTouchEvent，如果没有任何Behavior拦截，那么就会遍历调用onTouchEvent，直到返回true
    public boolean onTouchEvent(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull MotionEvent ev) {
        return false;
    }

    // 和getScrimOpacity配合设置关联view的蒙层绘制，会影响下层的其他view的可交互性。使用较少
    @ColorInt
    public int getScrimColor(@NonNull CoordinatorLayout parent, @NonNull V child) {
        return Color.BLACK;
    }
    
    @FloatRange(from = 0, to = 1)
    public float getScrimOpacity(@NonNull CoordinatorLayout parent, @NonNull V child) {
        return 0.f;
    }
    
    public boolean blocksInteractionBelow(@NonNull CoordinatorLayout parent, @NonNull V child) {
        return getScrimOpacity(parent, child) > 0.f;
    }
    
    // 在CoordinatorLayout的onMeasure开始，会先调用prepareChildren()，继而引发遍历此方法，返回true表示child对dependency有依赖关系，这里就会根据依赖关系对子view排序，测量和布局的过程中，都会依照这个顺序，而不是原本的顺序。
    // 在dependency被移除，或者发生滚动等情况，会引发onDependentViewChanged或者onDependentViewRemoved调用
    public boolean layoutDependsOn(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull View dependency) {
        return false;
    }
    
    public boolean onDependentViewChanged(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull View dependency) {
        return false;
    }
    
    public void onDependentViewRemoved(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull View dependency) {
    }
    
    // 在CoordinatorLayout测量此关联view前，先调用此方法，我们可以自行编写测量逻辑，返回true的话，CoordinatorLayout就不会再测量此view。也可以插入自己的其他逻辑，然后返回false，根据情况而定
    public boolean onMeasureChild(@NonNull CoordinatorLayout parent, @NonNull V child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        return false;
    }

    // 和onMeasureChild类似，只是针对于布局流程
    public boolean onLayoutChild(@NonNull CoordinatorLayout parent, @NonNull V child,
            int layoutDirection) {
        return false;
    }
    
    // 给view的LayoutParams设置一个tag对象。根据具体业务使用
    public static void setTag(@NonNull View child, @Nullable Object tag) {
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        lp.mBehaviorTag = tag;
    }
    
    // 获取tag
    @Nullable
    public static Object getTag(@NonNull View child) {
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();
        return lp.mBehaviorTag;
    }
    
    @Deprecated
    public boolean onStartNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes) {
        return false;
    }
    
    // 是否要处理CoordinatorLayout的嵌套滚动。在CoordinatorLayout自己的onStartNestedScroll方法中会遍历调用此方法，可以多个Behavior返回true。任何Behavior返回true的情况，都会记录到自己关联view的LayoutParams中。
    public boolean onStartNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes, @NestedScrollType int type) {
        if (type == ViewCompat.TYPE_TOUCH) {
            return onStartNestedScroll(coordinatorLayout, child, directTargetChild,
                    target, axes);
        }
        return false;
    }
    
    @Deprecated
    public void onNestedScrollAccepted(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes) {
        // Do nothing
    }
    
    // CoordinatorLayout的onNestedScrollAccepted中会遍历此方法
    public void onNestedScrollAccepted(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View directTargetChild, @NonNull View target,
            @ScrollAxis int axes, @NestedScrollType int type) {
        if (type == ViewCompat.TYPE_TOUCH) {
            onNestedScrollAccepted(coordinatorLayout, child, directTargetChild,
                    target, axes);
        }
    }
    
    @Deprecated
    public void onStopNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target) {
        // Do nothing
    }
    
    // 
    public void onStopNestedScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, @NestedScrollType int type) {
        if (type == ViewCompat.TYPE_TOUCH) {
            onStopNestedScroll(coordinatorLayout, child, target);
        }
    }
    
    @Deprecated
    public void onNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull V child,
            @NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
            int dyUnconsumed) {
        // Do nothing
    }
    
    @Deprecated
    public void onNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull V child,
            @NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
            int dyUnconsumed, @NestedScrollType int type) {
        if (type == ViewCompat.TYPE_TOUCH) {
            onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed,
                    dxUnconsumed, dyUnconsumed);
        }
    }
    
    // 发起嵌套滚动的view在滚动后，调用CoordinatorLayout的onNestedScroll方法，CoordinatorLayout会遍历所有Behavior的此方法，和onNestedPreScroll一样选择消费滚动距离的最大值作为自己的消费量
    public void onNestedScroll(@NonNull CoordinatorLayout coordinatorLayout, @NonNull V child,
            @NonNull View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
            int dyUnconsumed, @NestedScrollType int type, @NonNull int[] consumed) {
        // In the case that this nested scrolling v3 version is not implemented, we call the v2
        // version in case the v2 version is. We Also consume all of the unconsumed scroll
        // distances.
        consumed[0] += dxUnconsumed;
        consumed[1] += dyUnconsumed;
        onNestedScroll(coordinatorLayout, child, target, dxConsumed, dyConsumed,
                dxUnconsumed, dyUnconsumed, type);
    }
    
    @Deprecated
    public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, int dx, int dy, @NonNull int[] consumed) {
        // Do nothing
    }
    
    // CoordinatorLayout自身onNestedPreScroll被调用时，也就是发起嵌套滚动的view即将滚动，会遍历所有Behavior的onNestedPreScroll，并以其中消费滚动距离的最大值作为自己的消费量，报告给发起嵌套滚动的view
    public void onNestedPreScroll(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, int dx, int dy, @NonNull int[] consumed,
            @NestedScrollType int type) {
        if (type == ViewCompat.TYPE_TOUCH) {
            onNestedPreScroll(coordinatorLayout, child, target, dx, dy, consumed);
        }
    }
    
    // CoordinatorLayout的onNestedFling会遍历所有接受嵌套滚动的Behavior的onNestedFling
    public boolean onNestedFling(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, float velocityX, float velocityY,
            boolean consumed) {
        return false;
    }
    
    // 同上
    public boolean onNestedPreFling(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull View target, float velocityX, float velocityY) {
        return false;
    }
    
    // CoordinatorLayout回调OnApplyWindowInsetsListener时，会调用
    @NonNull
    public WindowInsetsCompat onApplyWindowInsets(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull WindowInsetsCompat insets) {
        return insets;
    }
    
    // CoordinatorLayout被调用时，转发到这里
    public boolean onRequestChildRectangleOnScreen(@NonNull CoordinatorLayout coordinatorLayout,
            @NonNull V child, @NonNull Rect rectangle, boolean immediate) {
        return false;
    }

    // 和onSaveInstanceState一起，在CoordinatorLayout的同名方法被调用时，转发到Behavior
    public void onRestoreInstanceState(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull Parcelable state) {
        // no-op
    }
    
    @Nullable
    public Parcelable onSaveInstanceState(@NonNull CoordinatorLayout parent, @NonNull V child) {
        return BaseSavedState.EMPTY_STATE;
    }
    
    // 配合view设置的layout_insetEdge来使用
    public boolean getInsetDodgeRect(@NonNull CoordinatorLayout parent, @NonNull V child,
            @NonNull Rect rect) {
        return false;
    }
}
```
看完这些方法，以及对照CoordinatorLayout中是怎么调用它们的，就知道CoordinatorLayout作为一个容器，把接收到的各种事件都转发到Behavior中。Behavior的方法，都有默认实现，我们可以选择有必要的部分重写。

## CoordinatorLayout的使用
CoordinatorLayout作为一个ViewGroup，也就是一个布局控件，除了配合Behavior使用，它还像普通的布局控件一样，在自定义的LayoutParams中提供了一些布局参数，这样它内部的子View可以使用一些布局参数。

* layout_behavior 这个属性就是用来在xml中去给view设置关联的Behavior。（实现CoordinatorLayout.AttachedBehavior接口的view，可以自带默认的Behavior）
* layout_gravity 设置view相对于CoordinatorLayout的位置，和FrameLayout类似
* layout_anchor和layout_anchorGravity 可以设置view相对于另一个view布局，CoordinatorLayout会考虑Behavior中的layoutDependsOn以及这里的layout_anchor，一起对依赖顺序
* keylines和layout_keyline keylines是CoordinatorLayout自己使用的，设置一个整数数组的资源，子view使用layout_keyline属性设置索引，例如1，就会使用数组资源的第二个数值作为自己的水平位置偏移，在布局时根据该值调整位置
* layout_insetEdge和layout_dodgeInsetEdges 这两个属性是配合使用，比如A设置layout_insetEdge为bottom，B设置layout_dodgeInsetEdges为bottom，则B会避免出现在A的下方，即使有margin等因素，也会被强制修改布局位置来符合这个要求

