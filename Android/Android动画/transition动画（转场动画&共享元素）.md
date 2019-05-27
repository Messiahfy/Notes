## 1.概述
&#160; &#160; &#160; &#160;`Android`的过渡动画（`transition`）是在`android 4.4`（有部分`transition`效果是`Android 5.0`引入的）版本引入的，`transition`框架可以通过简单地提供**起始布局**和**结束布局**来执行UI从起始到结束的过渡动画（内部使用了属性动画）。 可以选择所需的过渡动画类型（例如淡入/淡出或更改视图大小），`transition`框架会计算出如何从起始布局使用动画变化到结束布局。其中有几个关键：
* `Scene`：`Scene`（场景）引用了布局对象或资源id，表示起始或结束布局的状态，相当于属性动画中的起始值和结束值。
* `Transition`：表示在起始和结束的间过渡的动画，如自带的子类`slide`等，内部会通过`Scene`获取`sceneRoot`（`Scene`的根布局）从而得到起始或结束的视图层次的各种属性，通过起始和结束的各种属性值来执行具体动画。
* `TransitionManager`：用于切换`Scene`并执行设定的`Transition`动画。

**核心流程：**
1. 为初始和结束的布局构造`Scene`实例，但大多数情况只需构造结束的`Scene`，初始的`Scene`自动设为当前布局。
2. 构造`transition`实例来设置想要的动画类型。
3. 使用`TransitionManager`的某些方法来切换布局并执行动画。
---------------------------------------------------
**一个基本例子：**点击按钮1后，交换了1和2的位置。
![简单示例](https://upload-images.jianshu.io/upload_images/3468445-a9b7127b70dd28a3.gif?imageMogr2/auto-orient/strip)
`Activity`的布局文件：
```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/scene_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <Button
        android:id="@+id/button1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="111" />

    <Button
        android:id="@+id/button2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="222" />
</LinearLayout>
```
`scene2`的布局文件`R.layout.scene2`：
```
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">

    <Button
        android:id="@+id/button2"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="222" />

    <Button
        android:id="@+id/button1"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="111" />
</merge>
```
`Activity`的`onCreate()`方法中的代码：
```
@Override
    protected void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        viewGroup = findViewById(R.id.scene_root);
        button1 = findViewById(R.id.button1);
        //构造场景2，以根布局为sceneRoot，以R.layout.scene2为场景2的布局。
        scene2 = Scene.getSceneForLayout(viewGroup, R.layout.scene2, this);
        button1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //点击后调用TransitionManager.go方法，将场景切换到scene2，并使用
                //transition的子类ChangeBounds来设置动画；
                //方法内部会调用scene2.enter()方法，会让根布局removeAllViews删除所有 
                //子view，然后把scene2中设置的布局添加到根布局
                TransitionManager.go(scene2, new ChangeBounds());
            }
        });
    }
```
&#160; &#160; &#160; &#160;`Transition`（及其子类）会通过对比**起始布局**和**结束布局**中的相同`id`的`View`，来对其执行动画，即使`View`类型变了但只要`id`相同即认为是同一个`View`。**但是**例如是`fade`消失出现之类的效果，初始和结束场景中并没有对应的`view`，这类情况有对应`id`的就不会执行动画，因为认为没有消失和出现。
&#160; &#160; &#160; &#160;在上面的例子中，使用了`ChangeBounds`动画，只要位置或者大小变化就有动画效果，这里`场景2`的`button`交换了位置，所以对`button1`和`button2`分别执行了移动到新位置的动画。
> 可以看出，在对多个`View`执行动画时，如果不用`transition`动画，则**要么**对每个`View`都分别设置动画，这样比较麻烦；**要么**对所有`View`嵌套一个布局，但这样会冗余一个布局，且所有的`View`（包括布局中的其他不想执行动画的`View`）会执行相同的动画，比如都向上移动，而不能像上面例子那样分别向上和向下移动，又或者在两个`View`不直接在同一个`ViewGroup`中时，嵌套一层布局并不一定适用。

## 2.API介绍
`Scene`类：
| 方法 | 描述 |
|--|--|
|Scene(ViewGroup sceneRoot)|构造函数，传入根布局，没有要转换的布局信息|
|Scene(ViewGroup sceneRoot, View layout)|构造函数，传入根布局和要转换的布局，转换时要转化的布局会加载到根布局中作为子view|
| enter() |将根布局的子view清除，并把设置的要转换的布局加载到根布局|
| exit() |源码中仅调用了mExitAction.run()|
| getSceneForLayout(ViewGroup sceneRoot, int layoutId, Context context) |**静态方法**，构造一个scene实例|
| getSceneRoot () | 返回设置的根布局 |
| setEnterAction(Runnable action) | 设置一个runnable，在enter()方法中被调用 |
| setExitAction(Runnable action) | 设置一个runnable，在exit()方法中被调用 |
在大多数情况下，设置`Scene`的`EnterAction`或`ExitAction`是不必要的，因为框架动画场景之间的变化是自动的。
`Scene`的`Action`一般用于处理如下情况：
* 执行动画的`views`不在同一个视图层级，可以在`action`中操作执行另一个`view`的动画
* 为`transition`框架不能对其自动执行动画的`views`执行动画，例如`ListView`的子`view`


`TransitionManager`类：
| 方法 | 描述 |
|--|--|
|beginDelayedTransition(final ViewGroup sceneRoot)|**静态方法**，使用此方法则不用构造`Scene`，调用此方法和下一帧渲染之间，对设置的sceneRoot布局中的View改变属性值就可以实现动画效果|
|beginDelayedTransition(final ViewGroup sceneRoot, Transition transition)|**静态方法**，同上，但上面方法用的默认的AutoTransition|
|endTransitions(final ViewGroup sceneRoot)|**静态方法**，结束挂起和正在进行的transition|
|go(Scene scene, Transition transition)|**静态方法**，切换场景为传入的scene|
|go(Scene scene)|**静态方法**，同上，但使用默认动画|
|setTransition(Scene scene, Transition transition)|实际是put一个键值对，在调用transitionTo(Scene scene)方法时，如果传入的就是在setTransition方法中传入的scene，就会使用对应的transition|
|setTransition(Scene fromScene, Scene toScene, Transition transition)|同上，但多了一个条件：初始的scene要是传入的fromScene，才会使用这个transition，这种情况就不能省略构造初始scene|
|transitionTo(Scene scene)|切换scene，和上面两个方法配合使用|

**transition留到自定义transition时再了解**

## 3.Transition
&#160; &#160; &#160; &#160;`transition`框架用`Transition`对象来表示`Scene`之间的动画风格，可以用内置的子类如`AutoTransition`、`Fade`或者自定义来实例化`transition`，然后结合`Scene`和`TransitionManager`来执行动画。
&#160; &#160; &#160; &#160;`transition`的生命周期回调可以通过`addListener(TransitionListener listener)`添加监听器。
#### 3.1 创建transition实例
1.在代码中创建，例如：
```
Transition mFadeTransition = new Fade();
```

2.从资源文件中创建
在`res/transition/`目录中（默认就是fade_in_out）
```
<?xml version="1.0" encoding="utf-8"?>
<fade xmlns:android="http://schemas.android.com/apk/res/android"
    android:fadingMode="fade_in_out" />
```
然后在代码中加载：
```
Transition mFadeTransition =
        TransitionInflater.from(this).
        inflateTransition(R.transition.fade_transition);
```
#### 3.2 使用transition
调用`TransitionManager `的`go`方法、`transitionTo`等

---------------------------------------------------------
#### 3.3 选择指定的目标views
`transition`默认应用于`scene`中的所有`view`，但在某些情况只想应用于`scene`中的部分`view`，通过`transition`的`addTarget`、`removeTarget`和`excludeTarget`可以选择或排除目标。例如将上半部分往上退出，下半部分往下退出，达到从中间分裂的效果，则可以使用一个`transitionSet`，内部的两个`transition`分别设置`target`为上下部分。

`addTarget`方法有**四个重载**：
**相同点**：默认情况下，`transition`对`scene`的**根布局**中的所有`view`起作用，调用`addTarget`方法后将只对`add`的`view`执行动画，**其他将被忽略**。（四个重载方法添加目标是添加到**各自**的`ArrayList`）
|方法|不同点描述|
|--|--|
|addTarget(View target)| 此方法传入视图的实例，要**注意**的是：两个场景中的相同`ID`的视图不是同一个实例，所以对目标执行动画时不会对结束场景中的相同`ID`的视图起作用；<br/><br/>想要都起作用就应该使用`addTarget(int targetId)`方法；<br/><br/>而`addTarget(View target)`方法在某些情况更加适应，如要将没有`ID`的视图设为目标，通过ViewGroup.getChildAt(index)等方法获取视图对象并传入`addTarget(View target)`|
|addTarget(int targetId)|设置`targetIds`会将`Transition`限制为仅监听具有这些`ID`的视图并对其进行操作。 具有不同`ID`或无`ID`的视图将被忽略。|
|addTarget(Class targetType)| 设置Class，即只要是或者继承自该类，就添加为目标。例如传入View.class，则对场景中的全部`view`都会起作用 |
|addTarget(String targetName)| 设置`transitionName`，只要是视图的`transitionName`属性是传入的值，就添加为目标。和`ID`的效果类似|

`removeTarget `方法**也有对应的四个重载**，只是注意`removeTarget(int targetId)`在`Android 6 `以下有把视图`ID`当做数组`index`从而导致数组越界的`bug`。

`excludeTarget`方法：和`removeTarget `方法不同的是，`removeTarget `方法是删除`addTarget`方法添加的目标；而`excludeTarget`是排除目标，无论它是否被`addTarget`添加。没有调用`addTarget`时，所有`view`都是目标，调用`addTarget`时，只有`add`的才是目标，`excludeTarget`都可以使用。

`excludeChildren`方法：和`excludeTarget`作用类似，不同的是排除的是指定目标的子`view`
#### 3.4 指定多个transition
&#160; &#160; &#160; &#160;在一些情况下，视图可能又显示和消失，也有移动位置等变化，所以需要使用`transitionSet`来指定多个`transition`，这里用资源文件的方式定义和`AutoTransition`一样效果的动画：
```
<transitionSet xmlns:android="http://schemas.android.com/apk/res/android"
    android:transitionOrdering="sequential"> //可以设置顺序执行还是同时执行
    <fade android:fadingMode="fade_out" />
    <changeBounds />
    <fade android:fadingMode="fade_in" />
</transitionSet>
```
然后使用`TransitionInflater.from(this).inflateTransition(R.transition.xxx)`来获得实例。
&#160; &#160; &#160; &#160;也可以在代码中使用`transitionSet.addTransition`方法来设置多个`transition`。
#### 3.5 使用transition而不用scene
如果不想因为一些细微的变化而创建两个布局文件，可以使用`TransitionManager.beginDelayedTransition()`方法。
1. 调用此方法，传入`ViewGroup`（和`transition`），过渡动画框架会保存当前`ViewGroup`中的子`view`的状态和属性值。
2. 在下一帧重绘的时候，会根据`view`的改变来执行动画

## 4. 限制
* 过渡动画应用于`SurfaceView`可能无法正常显示，因为`SurfaceView`没有运行于`UI Thread`，因此更新可能与其他视图不同步。
* 应用于TextureView时，某些特定的过渡类型可能无法产生所需的动画效果。
* 扩展AdapterView的类（如ListView）以与`transition`框架不兼容的方式管理其子视图。 如果您尝试基于AdapterView为视图设置动画，则device display may hang。
* 如果您尝试使用动画调整TextView的大小，文本将在对象完全调整大小之前弹出到新位置。 要避免此问题，请不要为包含文本的视图的大小调整设置动画。

## 5.自定义transition
自定义transition需要提供捕获属性值并生成动画的代码；您可能还想为动画定义目标视图的子集。
#### 5.1 继承Transition类
```
public class CustomTransition extends Transition {

    @Override
    public void captureStartValues(TransitionValues values) {}

    @Override
    public void captureEndValues(TransitionValues values) {}

    @Override
    public Animator createAnimator(ViewGroup sceneRoot,
                                   TransitionValues startValues,
                                   TransitionValues endValues) {}
}
```
下面解释如何重写这些方法。
#### 5.2 捕获view的属性值
&#160; &#160; &#160; &#160;`transition`动画使用了属性动画，属性动画在指定的时间段内更改起始值和结束值之间的视图属性，因此`transition`框架需要同时具有属性的起始值和结束值以构造动画。
&#160; &#160; &#160; &#160;但是，属性动画通常只需要所有视图属性值的一小部分。 例如，颜色动画需要颜色属性值，而移动动画需要位置属性值。 由于对于`transition`来说动画所需的属性值是特定的，因此`transition`框架不会为`transition`提供每个属性值。 相反，框架调用回调函数，允许`transition`仅捕获它需要的属性值并将它们存储在框架中。
-------------------------------------------
1. **捕获起始值**
&#160; &#160; &#160; &#160;要将view的起始属性值传递给框架，请实现`captureStartValues(transitionValues)`方法。`transition`框架（`transition`的`captureValues`方法）会为起始`scene`中的每个`view`都调用`captureStartValues(transitionValues)`方法。参数是一个`TransitionValues`对象，它包含对`view`的引用，以及一个`Map`实例，您可以在其中存储所需的`view`属性值，存储的这些属性值均会被`transition`框架获取（实际就是保存在`transition`这个抽象父类的实例域中，当然也就是自定义的这个子类的实例域）。
&#160; &#160; &#160; &#160;为了确保属性值的`key`不会和`TransitionValues`的其他`key`冲突，例如继承slide，，请使用以下命名方案：
> package_name:transition_name:property_name
实现`captureStartValues()`的官网示例代码如下：
```
public class CustomTransition extends Transition {

    //使用语法package_name:transition_class:property_name定义存储属性值的key，以避免冲突
    private static final String PROPNAME_BACKGROUND =
            "com.example.android.customtransition:CustomTransition:background";

    @Override
    public void captureStartValues(TransitionValues transitionValues) {
        // 在captureValues方法中处理，因为和捕获结束值的代码类似，所以抽取到
        // captureValues方法中执行
        captureValues(transitionValues);
    }


    // 对于 transitionValues.view 中的view, 获取想要的属性值并把它们 
    // 放入到 transitionValues.values 键值对中
    private void captureValues(TransitionValues transitionValues) {
        // 获得view的引用
        View view = transitionValues.view;
        // 存储它的background属性到键值对中
        transitionValues.values.put(PROPNAME_BACKGROUND, view.getBackground());
    }
    ...
}
```
2. **捕获结束值**
&#160; &#160; &#160; &#160;框架为结束`scene`中的每个目标`view`调用一次captureEndValues(TransitionValues)函数。一般情况，`captureEndValues()`与`captureStartValues()`的工作方式相同。
```
@Override
public void captureEndValues(TransitionValues transitionValues) {
    captureValues(transitionValues);
}
```
-------------------------------------
在上面示例中，`captureStartValues()`和`captureEndValues()`函数都调用`captureValues()`来检索和存储值。 `captureValues()`检索的`view`属性是相同的，但它**在起始和结束场景中具有不同的值**。 该框架为`view`的起始和结束状态维护两个单独的键值对。
#### 5.3 创建animator
&#160; &#160; &#160; &#160;在捕获了起始值和结束值后，还要重写`createAnimator()`方法来提供`animator`。框架调用此方法时，会传递`scene`的`根视图`和包含您捕获的`起始值`和`结束值`的`TransitionValues`对象。
&#160; &#160; &#160; &#160;框架调用`createAnimator()`方法的次数取决于开始和结束`scene`之间发生的更改。例如**淡出/淡入动画**。如果**起始场景**有**五个目标**，其中**两个**从**结束场景**中移除，并且**结束场景**具有来自**起始场景**的三个目标加上一个新目标，则框架调用`createAnimator()`**六次**：三次调用对停留在两个场景对象中的目标执行淡出和淡入动画（两个场景都存在则起始值和初始值相同，看不出淡入/淡出效果，但是还是会调用`createAnimator()`方法）；另外两个调用对从结束场景中删除的目标的执行淡出;另一个一个调用对结束场景中新目标执行淡入。
&#160; &#160; &#160; &#160;对于存在于开始和结束场景的目标视图，框架分别对**起始值**和**结束值**都提供了`TransitionValues`对象。对于仅存在于开始或结束场景中的目标视图，框架为相应的参数提供`TransitionValues`对象，为另一个提供`null`。
[官方参考示例](https://github.com/googlesamples/android-CustomTransition/blob/master/Application/src/main/java/com/example/android/customtransition/ChangeColor.java)

## 6.transition用于Activity/Fragment的转场效果
可以为**入场**、**退场**、和**共享元素**执行过渡动画

对于**入场**和**退场**的过渡动画，只要是继承自`Visibility`的都可以，比如系统自带的`explode`、`slide`和`fade`。

-------------------------
对于**共享元素**，系统支持以下过渡动画：
* `changeBounds` ：检测`view`的位置边界创建移动和缩放动画
* `changeClipBounds `：检测`view`的剪切区域的位置边界，和`ChangeBounds`类似。不过`ChangeBounds`针对的是`view`而`ChangeClipBounds`针对的是`view`的剪切区域(`setClipBound(Rect rect)` 中的`rect`)。如果没有设置则没有动画效果
* `changeTransform `：检测`view`的`scale`和`rotation`创建缩放和旋转动画
* `changeImageTransform `：检测`ImageView`（这里是专指`ImageView`）的尺寸，位置以及`ScaleType`，并创建相应动画。
-------------------------
#### 6.1 检查系统版本
`transition`动画用于转场效果的`API`必须在`android 5.0及以上`
#### 6.2 指定transition
1. 启用`activity`的`transition`转场
在`style`中添加（`Material`主题自动设置为`true`）
```
<!-- 开启 window content transitions -->
<item name="android:windowActivityTransitions">true</item>
```
或者在`activity`的`setContentView`前调用`getWindow().requestFeature(Window.FEATURE_CONTENT_TRANSITIONS);`

2.  设置相应的`transition`

![从A启动B](https://upload-images.jianshu.io/upload_images/3468445-96bbb5b07df50a62.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![从B返回A](https://upload-images.jianshu.io/upload_images/3468445-d55202049919e3b5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


可在主题的`xml`中设置
```
<item name="android:windowEnterTransition">@android:transition/slide_right</item>
<item name="android:windowExitTransition">@android:transition/slide_left</item>
<item name="android:windowReturnTransition">@android:transition/slide_right</item>
<item name="android:windowReenterTransition">@android:transition/slide_left</item>
```
也可以在`activity`的代码中设置（`activity`属于`A`还是`B`是相对的，所以有些情况需要四个都设置）
```
//这两个在A中设置
getWindow().setExitTransition(new Slide(Gravity.LEFT));
getWindow().setReenterTransition(new Slide(Gravity.LEFT));
//这两个在B中设置
getWindow().setEnterTransition(new Slide(Gravity.RIGHT));
getWindow().setReturnTransition(new Slide(Gravity.RIGHT));
```
3. 启动activity
```
startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this).toBundle());
```
> 转场的时候，activity A和B的动画是一起执行的，如果想改成前一个activity转场动画结束再执行另一个，可以调用`getWindow().setAllowEnterTransitionOverlap(false)`，也有`getWindow().setAllowReturnTransitionOverlap(false)`

> 如果要排除某些view，例如状态栏，可以调用`transition.excludeTarget(android.R.id.statusBarBackground,true)`，transition的xml定义方式中也有<targets>属性可以设置

**如果是`Fragment`**：方式与`Activity`类似，只是不用`getWindow()`，而是直接调用。
如`setEnterTransition(transition)`
##7. 分享元素
要在具有共享元素的两个`Activity`之间制作屏幕过渡动画：
1. **开启功能**：在主题中或Activity的代码中启用窗口内容转换，如上面的转场。
2. **设置transition**：在代码中调用`getWindow().setSharedElementExitTransition(transition)`或者在主题`xml`中设置`android:windowSharedElementEnterTransition`属性（包括`exit`、`return`和`reenter`）
4. **指定transitionName**：使用`android:transitionName`属性为两个布局`XML`中的共享元素指定一个公用名。也可以使用`View.setTransitionName`方法在代码中分别设置
5. 使用`ActivityOptions.makeSceneTransitionAnimation()`函数。此方法有两个重载：
其一：`makeSceneTransitionAnimation(Activity activity, View sharedElement, String sharedElementName)`这个方法专用于一个共享元素；
其二：`makeSceneTransitionAnimation(Activity activity, Pair<View, String>... sharedElements)`这个方法用于任意个共享元素

要在`Fragment`间使用的话：
1. **设置transition**：`setSharedElementEnterTransition()`和` setSharedElementReturnTransition()`
2. **指定transitionName**：`FragmentTransaction.addSharedElement`方法
```
getSupportFragmentManager()
        .beginTransaction()
        .addSharedElement(sharedElement, transitionName)
        //.add、remove、replace、pop等操作
```

## 8.设置动画路径
使用`PathMotion`的子类设置动画路径，待后面再细看**********************************？？？？？？？？？？？？？？？？？？？