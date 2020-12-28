## 前言
在Flutter引擎启动部分，以Android平台为例，看到了执行Dart的main方法的过程，这里从Dart部分的main方法开始，分析Flutter应用层的启动流程。

## 代码分析
`main`方法会调用`runApp`，传入根widget，根widget会强制铺满屏幕：
```
void main() => runApp(MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
        // ...
    );
  }
}
```

`runApp`方法如下：
```
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```
这里执行了三步代码，我们分别进行分析。

### ensureInitialized
首先执行`WidgetsFlutterBinding`的`ensureInitialized`方法：
```
class WidgetsFlutterBinding extends BindingBase with GestureBinding, SchedulerBinding, ServicesBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```
这里就是简单调用了`WidgetsBinding`的构造函数，但是重要的是，`WidgetsFlutterBinding`继承了`BindingBase`，并且使用`with`混入多个`mixin`类型的Binding类。这里会先调用父类BindingBase的构造函数：
```
BindingBase() {
  //...
  initInstances();
  // 注册各种扩展用于调试，不具体分析
  initServiceExtensions();
  //...
}
```
构造函数中主要是调用了两个方法，在看先这两个方法前，先解释一下`with`和`mixin`的细节。`with`多个类，相当于连续继承，对于同一方法，`with`后的类，越后面的会覆盖前面的类中的方法，这是`mixin`的多态表现形式。

那么在这里，会执行最后也就是`WidgetsBinding`的`initInstances()`，但是其中又会先执行`super.initInstances()`，并且前面的每个类也是如此，所以最终又变为从`GestureBinding`的`initInstances()`开始执行。

第一个是`GestureBinding`
```
void initInstances() {
  super.initInstances(); // BindingBase的initInstances()只是执行断言，可忽略
  _instance = this;
  window.onPointerDataPacket = _handlePointerDataPacket;
}
```
主要就是给`window`设置手势事件的回调

第二个是`SchedulerBinding`
```
void initInstances() {
  super.initInstances();
  _instance = this;

  if (!kReleaseMode) {
    int frameNumber = 0;
    addTimingsCallback((List<FrameTiming> timings) {
      for (final FrameTiming frameTiming in timings) {
        frameNumber += 1;
        _profileFramePostEvent(frameNumber, frameTiming);
      }
    });
  }
}
```
运行帧绘制相关回调的调度者，比如动画就会用到它

第三个是`ServicesBinding`
```
void initInstances() {
  super.initInstances();
  _instance = this;
  _defaultBinaryMessenger = createBinaryMessenger();
  _restorationManager = createRestorationManager();
  window.onPlatformMessage = defaultBinaryMessenger.handlePlatformMessage;
  initLicenses();
  SystemChannels.system.setMessageHandler((dynamic message) => handleSystemMessag(message as Object));
  SystemChannels.lifecycle.setMessageHandler(_handleLifecycleMessage);
  readInitialLifecycleStateFromNativeWindow();
}
```
设置Platform Message相关的监听和处理

第四个是`PaintingBinding`
```
void initInstances() {
  super.initInstances();
  _instance = this;
  _imageCache = createImageCache();
  shaderWarmUp?.execute();
}
```
创建图片缓存

第五个是`SemanticsBinding`
```
void initInstances() {
  super.initInstances();
  _instance = this;
  _accessibilityFeatures = window.accessibilityFeatures;
}
```
辅助功能相关

第六个是`RendererBinding`
```
void initInstances() {
  super.initInstances();
  _instance = this;
  // 1
  _pipelineOwner = PipelineOwner(
    onNeedVisualUpdate: ensureVisualUpdate,
    onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
    onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
  );
  // 2
  window
    ..onMetricsChanged = handleMetricsChanged
    ..onTextScaleFactorChanged = handleTextScaleFactorChanged
    ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged
    ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
    ..onSemanticsAction = _handleSemanticsAction;
  // 3
  initRenderView();
  _handleSemanticsEnabledChanged();
  assert(renderView != null);
  // 4
  addPersistentFrameCallback(_handlePersistentFrameCallback);
  initMouseTracker();
}
```
1. 创建`PipelineOwner`，管理渲染流水线
2. 给`window`设置各种回调，比如屏幕尺寸、密度等变化
3. 创建并初始化`RenderView`，下一帧将被绘制。`RenderView`就是`RenderObject`绘制树的根。initRenderView函数内部赋值renderView，而renderView重写了setter，会引发全部`RenderObject`调用`attach`
4. 向添加`SchedulerBinding`的`_persistentCallbacks`中添加回调，回调内容为执行渲染流水线

第七个是`WidgetsBinding`
```
void initInstances() {
  super.initInstances();
  _instance = this;
  //...
  // 1
  _buildOwner = BuildOwner();
  // 2
  buildOwner.onBuildScheduled = _handleBuildScheduled;
  // 3
  window.onLocaleChanged = handleLocaleChanged;
  window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
  // 4
  SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
  FlutterErrorDetails.propertiesTransformers.add(transformDebugCreator);
}
```
`WidgetsBinding`是`Widget`和flutter engine的桥梁。这里做了以下几步：
1. 创建`BuildOwner`，用于管理Widget框架，包括Element
2. 设置Element标记为dirty时，每次构建过程的回调，也就是渲染流程
3. 给window设置一些回调
4. 设置处理路由相关函数

`RendererBinding`中创建的`PipelineOwner`和这里的`BuildOwner`，是关于渲染流程的关键类

### scheduleAttachRootWidget
`runApp`函数中，执行的第二步就是`scheduleAttachRootWidget`
```
void scheduleAttachRootWidget(Widget rootWidget) {
  Timer.run(() {
    attachRootWidget(rootWidget);
  });
}
```
将调度到下一个消息循环中去执行`attachRootWidget`
```
void attachRootWidget(Widget rootWidget) {
  _readyToProduceFrames = true;
  _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
    container: renderView,
    debugShortDescription: '[root]',
    child: rootWidget,
  ).attachToRenderTree(buildOwner, renderViewElement as RenderObjectToWidgetElement<RenderBox>);
}
```
这里传入的`rootWidget`就是我们写的根Widget，`renderView`就是在`RendererBinding`中创建的`RenderView`，继承自`RenderObject`。因为渲染流程是以`Element`树为核心，`RenderObject`（`RenderView`）需要有对应的`element`，所以要在这里完成关联。

**`WidgetsBinding`将持有`renderViewElement`，并且关联了`RendererBinding`中创建的`RenderView`**

我们先来看`RenderObjectToWidgetAdapter`
```
class RenderObjectToWidgetAdapter<T extends RenderObject> extends RenderObjectWidget {
  RenderObjectToWidgetAdapter({
    this.child,
    this.container,
    this.debugShortDescription,
  }) : super(key: GlobalObjectKey(container));
  
  ...

  @override
  RenderObjectToWidgetElement<T> createElement() => RenderObjectToWidgetElement<T>(this);

  @override
  RenderObjectWithChildMixin<T> createRenderObject(BuildContext context) => container;

  RenderObjectToWidgetElement<T> attachToRenderTree(BuildOwner owner, [ RenderObjectToWidgetElement<T> element ]) {
    if (element == null) {
      owner.lockState(() {
        element = createElement(); // 创建Element
        assert(element != null);
        element.assignOwner(owner); // 指定BuildOwner
      });
      owner.buildScope(element, () { //调用回调，并build所有标记为dirty的element
        element.mount(null, null);
      });
      // This is most likely the first time the framework is ready to produce
      // a frame. Ensure that we are asked for one.
      SchedulerBinding.instance.ensureVisualUpdate();
    } else {
      element._newWidget = this;
      element.markNeedsBuild();
    }
    return element;
  }
}
```
省略了其他属性和方法。调用`attachToRenderTree`，就会调用`createElement`，这里就会创建`RenderObjectToWidgetElement`，并且指定了`BuildOwner`。然后调用回调，也就是`element.mount`，将把`RenderObjectToWidgetElement`加入到`Element`树中，并调用build，引发`rootWidget`的widget树递归构造Element树和RenderObject树。然后会build所有标记为dirty的element，第一次调用应该还没有标记为dirty的element。

可见`scheduleAttachRootWidget`流程的关键，就是把根Widget(`RenderObjectToWidgetAdapter`)和`RenderView`关联起来，并且把我们的`rootWidget`加入到了根Widget树中，一起构造了Widget、Element、RenderObject三颗树。

### scheduleWarmUpFrame
然后到最后一步，将调度执行渲染

```
void scheduleWarmUpFrame() {
  if (_warmUpFrame || schedulerPhase != SchedulerPhase.idle)
    return;
  _warmUpFrame = true;
  Timeline.startSync('Warm-up frame');
  final bool hadScheduledFrame = _hasScheduledFrame;
  // We use timers here to ensure that microtasks flush in between.
  Timer.run(() {
    // 调度执行帧准备绘制的方法
    handleBeginFrame(null);
  });
  Timer.run(() {
    // 调度执行帧绘制的方法
    handleDrawFrame();
    resetEpoch();// 热重载相关
    _warmUpFrame = false;
    if (hadScheduledFrame)
      scheduleFrame();
  });
  // Lock events so touch events etc don't insert themselves until the
  // scheduled frame has finished.
  lockEvents(() async {
    await endOfFrame;
    Timeline.finishSync();
  });
}
```
`handleBeginFrame`会执行`_transientCallbacks`中的回调函数，动画中的`Ticker`就会给`_transientCallbacks`中设置回调函数

`handleDrawFrame`如下：
```
void handleDrawFrame() {
  assert(_schedulerPhase == SchedulerPhase.midFrameMicrotasks);
  Timeline.finishSync(); // end the "Animate" phase
  try {
    // 执行_persistentCallbacks中的回调函数
    _schedulerPhase = SchedulerPhase.persistentCallbacks;
    for (final FrameCallback callback in _persistentCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp!);
    // 执行_postFrameCallbacks中的回调函数
    _schedulerPhase = SchedulerPhase.postFrameCallbacks;
    final List<FrameCallback> localPostFrameCallbacks =
        List<FrameCallback>.from(_postFrameCallbacks);
    // _postFrameCallbacks中的回调函数用完就清空
    _postFrameCallbacks.clear();
    for (final FrameCallback callback in localPostFrameCallbacks)
      _invokeFrameCallback(callback, _currentFrameTimeStamp!);
  } finally {
    _schedulerPhase = SchedulerPhase.idle;
    Timeline.finishSync(); // end the Frame
    //...
    _currentFrameTimeStamp = null;
  }
}
```
在前面分析的`RendererBinding`的`initInstances`函数中，已经向`_persistentCallbacks`中添加了`_handlePersistentFrameCallback`函数，现在就会执行该函数：
```
// RendererBinding

void _handlePersistentFrameCallback(Duration timeStamp) {
  drawFrame();
  _scheduleMouseTrackerUpdate();
}

void drawFrame() {
  assert(renderView != null);
  pipelineOwner.flushLayout();
  pipelineOwner.flushCompositingBits();
  pipelineOwner.flushPaint();
  if (sendFramesToEngine) {
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
    _firstFrameSent = true;
  }
}
```
`drawFrame()`函数是关键，它将完成整个渲染流水线。接下来我们对这里面的几步做分析：

首先看`pipelineOwner.flushLayout()`：
```
  void flushLayout() {
    //...
    try {
      // 遍历所有RenderObject
      while (_nodesNeedingLayout.isNotEmpty) {
        final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
        _nodesNeedingLayout = <RenderObject>[];
        for (final RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
          if (node._needsLayout && node.owner == this)
            // 有必要则执行布局流程
            node._layoutWithoutResize();
        }
      }
    } finally {
      //...
    }
  }
```
在`RendererBinding`的`initInstances`方法中，会调用`initRenderView`，其中会把`RenderView`添加到`_nodesNeedingLayout`列表中。初始应该只有`RenderView`在列表中。
```
  void _layoutWithoutResize() {
    //...
    try {
      performLayout();
      markNeedsSemanticsUpdate();
    } catch (e, stack) {
      //...
    }
    //...
    _needsLayout = false;
    markNeedsPaint();
  }
```
* `performLayout()`由`RenderObject`自行重写实现，其中会设置自己的大小，如果有子节点，那么还需要调用子节点的`layout`方法，引发整个树的布局
* `markNeedsPaint()`将当前`RenderObject`添加到`PipelineOwner`的`_nodesNeedingPaint`列表中，待绘制时使用

第二步是`pipelineOwner.flushCompositingBits()`，刷新合成标志，不了解这部分，略过

第三步是`pipelineOwner.flushPaint()`
```
  void flushPaint() {
    //...
    try {
      final List<RenderObject> dirtyNodes = _nodesNeedingPaint;
      _nodesNeedingPaint = <RenderObject>[];
      // Sort the dirty nodes in reverse order (deepest first).
      for (final RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => b.depth - a.depth)) {
        assert(node._layer != null);
        if (node._needsPaint && node.owner == this) {
          if (node._layer!.attached) {
            PaintingContext.repaintCompositedChild(node);
          } else {
            node._skippedPaintingOnLayer();
          }
        }
      }
      assert(_nodesNeedingPaint.isEmpty);
    } finally {
      //...
    }
  }
```
对于需要绘制的`RenderObject`，会调用`repaintCompositedChild`，其中会调用到`RenderObject`的`_paintWithContext`函数：
```
void _paintWithContext(PaintingContext context, Offset offset) {
  if (_needsLayout)
    return;
  //...
  RenderObject? debugLastActivePaint;
  //...
  _needsPaint = false;
  try {
    paint(context, offset);//调用paint
    //...
  } catch (e, stack) {
    _debugReportException('paint', e, stack);
  }
  //...
}
```
核心就是调用了`paint`方法，`RenderObject`子类将重写`paint`方法，在其中完成绘制工作（`PaintingContext`中可以获得canvas）。

`renderView.compositeFrame()`和`pipelineOwner.flushSemantics()`会使`Window`调用native方法，暂不分析

## 总结
综上分析，并结合flutter官方的渲染流水线，得出以下几点：

1. Animate: 遍历执行_transientCallbacks中的回调函数，动画相关的`Ticker`会往这里面添加回调函数
2. Build: `buildScope`函数引发Widget、Element、RenderObject三棵树的构造
3. Layout: `flushLayout`引发`RenderObject`确定位置和尺寸
4. Paint: `flushPaint`引发绘制流程，绘制命令将存储在layer中，后面会发给底层处理