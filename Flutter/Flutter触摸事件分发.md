## 前言
Flutter支持触摸和鼠标事件，都属于Pointer事件，另外也支持按键事件和焦点处理。这里以触摸事件为例来分析事件分发和处理的流程。

## 事件处理源码分析
`GestureBinding`的`initInstances()`函数中给`window`设置了触摸事件的回调
```
window.onPointerDataPacket = _handlePointerDataPacket;

void _handlePointerDataPacket(ui.PointerDataPacket packet) {
  // 把PointerData转换为逻辑像素的PointerEvent数据，并添加到_pendingPointerEvents双端队列中
  _pendingPointerEvents.addAll(PointerEventConverter.expand(packet.data, window.devicePixelRatio));
  if (!locked)
    _flushPointerEventQueue();
}
```
以安卓为例，在`FlutterView`的`onTouchEvent`中会把事件发到Flutter引擎，然后再转发到Dart中，并会调用到`_handlePointerDataPacket`函数。收到了较为原始的`PointerData`事件，会结合设备的像素密度，转换为`PointerEvent`类型的事件。对于触摸事件，需要关注以下几个子类型：
* `PointerDownEvent`：手指按下事件
* `PointerMoveEvent`：手指移动事件
* `PointerUpEvent`：手指抬起事件
* `PointerCancelEvent`：触摸事件取消

```
void _flushPointerEventQueue() {
  //...
  while (_pendingPointerEvents.isNotEmpty)
    _handlePointerEvent(_pendingPointerEvents.removeFirst());
}
```
循环处理队列中的事件
```
void _handlePointerEvent(PointerEvent event) {
  HitTestResult? hitTestResult;
  if (event is PointerDownEvent || event is PointerSignalEvent) {
    //手指按下
    hitTestResult = HitTestResult();
    //执行命中测试，得到命中的控件集合
    hitTest(hitTestResult, event.position);
    if (event is PointerDownEvent) {
      // 存储在_hitTests中
      _hitTests[event.pointer] = hitTestResult;
    }
    //...
  } else if (event is PointerUpEvent || event is PointerCancelEvent) {
    // 抬起和取消，移除
    hitTestResult = _hitTests.remove(event.pointer);
  } else if (event.down) {
    // 手指滑动中（按下和滑动的down属性都是true）
    hitTestResult = _hitTests[event.pointer];
  }
  //...
  if (hitTestResult != null ||
      event is PointerHoverEvent ||
      event is PointerAddedEvent ||
      event is PointerRemovedEvent) {
    // 分发事件
    dispatchEvent(event, hitTestResult);
  }
}
```
* `HitTestResult`：存储`HitTestEntry`集合，用于事件分发
* `HitTestEntry`：存储`HitTestTarget`，`GestureBinding`、`RenderObject`都实现了`HitTestTarget`接口
* `HitTestTarget`：包含`handleEvent`函数

### 命中测试和事件分发
首先看手指按下时，调用`hitTest`函数，这里会执行`RendererBinding`重写的`hitTest`：
```
// RendererBinding#hitTest

void hitTest(HitTestResult result, Offset position) {
  renderView.hitTest(result, position: position);
  super.hitTest(result, position);//调用下面super，也就是会在添加完命中的控件后，最后添加GestureBinding自身到HitTestResult中
}

// GestureBinding#hitTest

void hitTest(HitTestResult result, Offset position) {
  result.add(HitTestEntry(this));
}
```
从`renderView`开始执行`hitTest`：
```
  bool hitTest(HitTestResult result, { required Offset position }) {
    if (child != null)
      child!.hitTest(BoxHitTestResult.wrap(result), position: position);
    result.add(HitTestEntry(this));
    return true;
  }
```
由于`renderView`的child，也就是我们写的根Widget对应的`RenderObject`是铺满屏幕的，所以这里直接传触摸事件的position，不需要考虑偏移。

```
// RenderBox

bool hitTest(BoxHitTestResult result, { required Offset position }) {
  if (_size!.contains(position)) {
    if (hitTestChildren(result, position: position) || hitTestSelf(position)) {
      result.add(BoxHitTestEntry(this, position));
      return true;
    }
  }
  return false;
}
```
一般我们写的widget对应的`RenderObject`是`RenderBox`的子类，这里看`RenderBox`的`hitTest`函数，将调用`hitTestChildren`或者`hitTestSelf`，其中一个返回true，就会把当前`RenderObject`包装为`HitTestEntry`添加到`HitTestResult`中。

那么先来看`hitTestChildren`，`RenderBox`中没有具体实现，因为有子节点的`RenderBox`才需要实现此函数。以`RenderShiftedBox`（`Padding`对应的`RenderPadding`的父类）为例：
```
bool hitTestChildren(BoxHitTestResult result, { required Offset position }) {
  if (child != null) {
    final BoxParentData childParentData = child!.parentData as BoxParentData;
    return result.addWithPaintOffset(
      offset: childParentData.offset,
      position: position,
      hitTest: (BoxHitTestResult result, Offset? transformed) {
        return child!.hitTest(result, position: transformed!);
      },
    );
  }
  return false;
}
```
可以看到，这里使用了child的offset偏移位置，传给`addWithPaintOffset`函数，内部就会使用position减去offset，得到transformed再传给child。child一般也是一个`RenderBox`的子类，这时又可以看到上面的`hitTest`函数，会使用`_size!.contains(position)`来判断触摸事件是否在自己的范围内，对于child，这个position已经是parent结合child的offset转换后传来的，所以就可以直接用于判断是否在范围内。

那么，可以看出，以上流程会遍历完`RenderObject`树，对于命中的控件，会按照树的底部到顶部，也就是先添加child，再添加parent的顺序，添加到`HitTestResult`中。

---------------

`PointerDownEvent`事件执行完命中测试后，将执行`dispatchEvent`：
```
void dispatchEvent(PointerEvent event, HitTestResult? hitTestResult) {
  assert(!locked);
  // No hit test information implies that this is a pointer hover or
  // add/remove event. These events are specially routed here; other events
  // will be routed through the `handleEvent` below.
  if (hitTestResult == null) {
    try {
      pointerRouter.route(event);
    } catch (exception, stack) {
      //...
    }
    return;
  }
  for (final HitTestEntry entry in hitTestResult.path) {
    try {
      entry.target.handleEvent(event.transformed(entry.transform), entry);
    } catch (exception, stack) {
      //...
    }
  }
}
```
hitTestResult不为空，就循环调用`HitTestEntry`的`target`的`handleEvent`函数，而这个`target`就是之前执行命中测试时，添加的命中的各个`RenderObject`，以及最后添加的`GestureBinding`。`RenderObject`以及子类`RenderBox`都是空实现`handleEvent`函数，也就是不处理事件，一般我们要处理事件，会使用`Listener`、`GestureDetector`（内部也会使用`Listener`）等Widget，对应的`RenderObject`子类就是`RenderPointerListener`，那么我们来看`RenderPointerListener`的`handleEvent`函数：
```
void handleEvent(PointerEvent event, HitTestEntry entry) {
  if (onPointerDown != null && event is PointerDownEvent)
    return onPointerDown!(event);
  if (onPointerMove != null && event is PointerMoveEvent)
    return onPointerMove!(event);
  if (onPointerUp != null && event is PointerUpEvent)
    return onPointerUp!(event);
  if (onPointerCancel != null && event is PointerCancelEvent)
    return onPointerCancel!(event);
  if (onPointerSignal != null && event is PointerSignalEvent)
    return onPointerSignal!(event);
}
```
就是把事件交给对应的函数处理，而这些函数就是使用`Listener`等Widget时传入的。

----------------

前面这部分描述是以`down`事件为例，那么我们再回到`_handlePointerEvent`函数中，分析`move`和`up/cancel`的事件处理方式。
1. 对于`move`：不再需要像`down`一样执行命中测试，而是直接使用`down`事件对应的`hitTestResult`，然后和`down`事件一样，分发处理
2. 对于`up/cancel`：同样得到`down`对应的`hitTestResult`，分发处理，但为删掉该`hitTestResult`，因为事件流程已结束

## 事件冲突处理
从前面的分析可以发现，只要通过了命中测试，控件就能收到分发来的事件，而且就是直接遍历，并没有事件冲突、拦截之类的处理。比如一个`parent`内有一个`child`，这两个widget都是`Listenr`的子widget，`parent`的面积比`child`大；手指触摸`child`，`parent`和`child`对应的`Listenr`都会收到事件，这也证明了没有事件冲突处理的事实。

如果在`hitTest`中返回false，当前控件就不会收到分发的事件，如下两个`Widget`就是通过控制其对应的`RenderObject`的`hitTest`函数来控制是否接收事件
1. IgnorePointer 忽略事件，包括它自己
2. AbsorbPointer 拦截事件，不会传递给子节点，但自己能收到

以上的方式，只是直接的控制不接收事件，不适合动态的判断谁来接收事件。比如`parent`和`child`都能接收事件，但点击`child`的时候，只应该响应`child`的点击事件，`parent`同理；这种情况是不适合用`IgnorePointer`之类的方式来处理的。

那么在Flutter如何像Android中那样，可以动态处理事件冲突呢？这就要使用到`GestureDetecotor`，`GestureDetecotor`可以识别多种手势，并且能处理事件冲突。下面我们来分析`GestureDetecotor`是如何做到可以处理事件冲突的。
```
//GestureDetector

Widget build(BuildContext context) {
  final Map<Type, GestureRecognizerFactory> gestures = <Type, GestureRecognizerFactory>{};
  if (...
      onTap != null ||
      ...省略其他监听器
  ) {
    // gestures是一个map数据，设置TapGestureRecognizer对应的构造工厂对象
    gestures[TapGestureRecognizer] = GestureRecognizerFactoryWithHandlers<TapGestureRecognizer>(
      // 第一个参数，用于构造TapGestureRecognizer
      () => TapGestureRecognizer(debugOwner: this),
      // 第二个参数，用于初始化TapGestureRecognizer
      (TapGestureRecognizer instance) {
        instance
          //...
          ..onTap = onTap
          //...省略其他监听器
      },
    );
  }
  //...省略其他手势
  return RawGestureDetector(
    gestures: gestures,
    behavior: behavior,
    excludeFromSemantics: excludeFromSemantics,
    child: child,
  );
}
```
以`onTap`，也就是点击事件的回调为例。使用`GestureDetecotor`的时候，设置了`onTap`，那么在这里的build中，会往`gestures`中设置`TapGestureRecognizer`对应的构造工厂。`TapGestureRecognizer`是``GestureArenaMember`的子类，表示参与手势竞技场的成员，具体作用在后面的分析中了解。这里的最后创建了`RawGestureDetector`

`RawGestureDetector`是一个`StatefulWidget`，所以我们看到它对应的`RawGestureDetectorState`，首先是`initState`函数：
```
// RawGestureDetector

@override
void initState() {
  super.initState();
  _semantics = widget.semantics ?? _DefaultSemanticsGestureDelegate(this);
  _syncAll(widget.gestures);
}
```
初始化状态中主要就是执行`_syncAll`：
```
// RawGestureDetector

void _syncAll(Map<Type, GestureRecognizerFactory> gestures) {
  final Map<Type, GestureRecognizer> oldRecognizers = _recognizers;
  _recognizers = <Type, GestureRecognizer>{};
  for (final Type type in gestures.keys) {
    // 缓存有的话，就用缓存的对象，否则创建
    _recognizers[type] = oldRecognizers[type] ?? gestures[type].constructor();
    // 初始化
    gestures[type].initializer(_recognizers[type]);
  }
  for (final Type type in oldRecognizers.keys) {
    if (!_recognizers.containsKey(type))
      // 数据更新后，旧数据中有，但是新的数据中没有的对象执行dispose回调，表示不再需要
      oldRecognizers[type].dispose();
  }
}
```
这里的`gestures`参数就是前面`GestureDetector`的`build`函数中设置的map。重点就是调用了之前设置的构造和初始化函数，把构造的对象放到了`_recognizers`中，初始化时给`TapGestureRecognizer`设置了`onTap`回调函数。

然后看到`RawGestureDetector`的`build`函数：
```
// RawGestureDetector

Widget build(BuildContext context) {
  Widget result = Listener(
    onPointerDown: _handlePointerDown,
    behavior: widget.behavior ?? _defaultBehavior,
    child: widget.child,
  );
  //...
  return result;
}
```

---------------

这里就发现`GestureDetecotor`的最终实现也是`Listener`，给`Listener`设置了`onPointerDown`回调，发生`down`事件就会执行`_handlePointerDown`：
```
// RawGestureDetectorState

void _handlePointerDown(PointerDownEvent event) {
  for (final GestureRecognizer recognizer in _recognizers.values)
    recognizer.addPointer(event);
}
```
遍历`_recognizers`中的对象，在我们的例子中，这里就是`TapGestureRecognizer`，执行它的`addPointer`函数（在父类`GestureRecognizer`中）：
```
//GestureRecognizer

void addPointer(PointerDownEvent event) {
  _pointerToKind[event.pointer] = event.kind; // 记录事件类型
  if (isPointerAllowed(event)) {
    addAllowedPointer(event); // 实际注册的函数
  } else {
    handleNonAllowedPointer(event);
  }
}
```
`addPointer`在`down`事件时调用，作用就是注册当前手指，继续看`TapGestureRecognizer`的父类中的`addAllowedPointer`函数：
```
//BaseTapGestureRecognizer

void addAllowedPointer(PointerDownEvent event) {
  assert(event != null);
  if (state == GestureRecognizerState.ready) {
    // 初始状态时，收到down，则记录
    _down = event;
  } 
  if (_down != null) {
    // 然后执行父类的函数
    super.addAllowedPointer(event);
  }
}
```
继续看`BaseTapGestureRecognizer`父类`PrimaryPointerGestureRecognizer`中的`addAllowedPointer`函数：
```
//PrimaryPointerGestureRecognizer

void addAllowedPointer(PointerDownEvent event) {
  startTrackingPointer(event.pointer, event.transform);
  if (state == GestureRecognizerState.ready) {
    // 初始状态，也就是收到down的时候，初始化相关属性，并且修改状态为possible
    state = GestureRecognizerState.possible;
    primaryPointer = event.pointer; //单点触控，即该手指，如果是多点触控，则为第一个手指
    initialPosition = OffsetPair(local: event.localPosition, global: event.position);
    if (deadline != null)
      _timer = Timer(deadline!, () => didExceedDeadlineWithEvent(event));
  }
}
```
这里主要是两个部分：
1. 执行`startTrackingPointer`
2. 初始化状态
那么我们继续看`startTrackingPointer`里面做了什么：
```
//OneSequenceGestureRecognizer

void startTrackingPointer(int pointer, [Matrix4? transform]) {
  // 注册路由，也就是手指ID对应的事件处理函数handleEvent（在PrimaryPointerGestureRecognizer中实现）
  GestureBinding.instance!.pointerRouter.addRoute(pointer, handleEvent, transform);
  _trackedPointers.add(pointer);// 记录追踪的手指ID
  _entries[pointer] = _addPointerToArena(pointer);  //添加手指ID到竞技场，并记录返回的GestureArenaEntry
}
```
1. 注册路由：记录当前手指ID和对应的`handleEvent`函数，存储到`PointerRouter`的`_routeMap`中
2. 记录追踪的手指ID到当前类`OneSequenceGestureRecognizer`中
3. 添加手指ID到竞技场

第一步的`handleEvent`函数在后面被调用的时候再分析，这里的重点是第三步，执行`_addPointerToArena`：
```
//OneSequenceGestureRecognizer

  GestureArenaEntry _addPointerToArena(int pointer) {
    if (_team != null)
      return _team!.add(pointer, this);
    return GestureBinding.instance!.gestureArena.add(pointer, this);
  }
```
主要是调用了`GestureArenaManager`的`add`函数
```
//GestureArenaManager

GestureArenaEntry add(int pointer, GestureArenaMember member) {
  final _GestureArena state = _arenas.putIfAbsent(pointer, () {
    return _GestureArena();
  });
  state.add(member);
  return GestureArenaEntry._(this, pointer, member);
} 
```
`GestureArenaManager`中存在`Map<int, _GestureArena>`类型的属性`_arenas`，表示每个手指对应一个手势竞技场`_GestureArena`，每个`_GestureArena`内包含多个`GestureRecognizer`。`member`参数就是当前示例的`TapGestureRecognizer`，这里就会把当前`TapGestureRecognizer`添加到该手指对应的`_GestureArena`中。

---------------
到此，我们来总结一下前面这一部分流程的具体作用。使用`GestureDetecotor`，根据设置的手势监听器，创建对应的`GestureRecognizer`（例如`TapGestureRecognizer`），然后收到`down`事件，就把当前手指和对应的`GestureRecognizer`添加到`GestureArenaManager`内的相关数据结构中。

* `GestureArenaMember`：手势竞技场的参与者
* `GestureRecognizer`：是`GestureArenaMember`的子类，也就是说，手势识别者就是手势竞技场的参与者
* `GestureArenaManager`：持有所有的手势竞技场参与者，决定获胜者

前面的流程，收到了down，分发事件到`GestureDetecotor`相关处理者。而在前面分析命中测试流程的时候，我们知道在处理完`widget`的命中测试后，最后会把`GestureBinding`也添加到`HitTestResult`中，也就是说，事件最后也会分发到`GestureBinding`的`handleEvent`函数：
```
void handleEvent(PointerEvent event, HitTestEntry entry) {
  // 1
  pointerRouter.route(event);
  if (event is PointerDownEvent) {
    // 2
    gestureArena.close(event.pointer);
  } else if (event is PointerUpEvent) {
    // 3
    gestureArena.sweep(event.pointer);
  } else if (event is PointerSignalEvent) {
    pointerSignalResolver.resolve(event);
  }
}
```
1. 前面在`startTrackingPointer`中向`pointerRouter`中注册了`PrimaryPointerGestureRecognizer`中的`handleEvent`函数，这里就会执行它：
```
// PrimaryPointerGestureRecognizer

void handleEvent(PointerEvent event) {
  if (state == GestureRecognizerState.possible && event.pointer == primaryPointer) {
    //...
    if (event is PointerMoveEvent && (isPreAcceptSlopPastTolerance || isPostAcceptSlopPastTolerance)) {
      // 如果是滑动事件，且滑动距离超过阈值，那么对于点击来说，视为取消，
      resolve(GestureDisposition.rejected);
      stopTrackingPointer(primaryPointer!);
    } else {
      // 对于down、up、cancle，将执行这里
      handlePrimaryPointer(event);
    }
  }
  stopTrackingIfPointerNoLongerDown(event); //如果是up或者cancle，就会取消注册的handleEvent函数
}
```
对于当前情况，还是处于`down`事件，所以继续分析`handlePrimaryPointer`：
```
// BaseTapGestureRecognizer

void handlePrimaryPointer(PointerEvent event) {
  if (event is PointerUpEvent) {
    _up = event;
    _checkUp();
  } else if (event is PointerCancelEvent) {
    resolve(GestureDisposition.rejected);
    if (_sentTapDown) {
      _checkCancel(event, '');
    }
    _reset();
  } else if (event.buttons != _down!.buttons) {
    resolve(GestureDisposition.rejected);
    stopTrackingPointer(primaryPointer!);
  }
}
```
对于`down`事件，在这里将不会做任何事情

2. 这里还是`down`事件流程，所以执行`gestureArena`的`close`函数：
```
void close(int pointer) {
  final _GestureArena? state = _arenas[pointer];
  if (state == null)
    return; // This arena either never existed or has been resolved.
  state.isOpen = false; // 不再接收新的竞技场成员
  _tryToResolveArena(pointer, state);
}
```
继续执行`_tryToResolveArena`：
```
void _tryToResolveArena(int pointer, _GestureArena state) {
  if (state.members.length == 1) {
    // 1.只有一个竞争成员，那么会直接调用该GestureArenaMember的acceptGesture
    scheduleMicrotask(() => _resolveByDefault(pointer, state));
  } else if (state.members.isEmpty) {
    // 2.如果没有成员，就跳过
    _arenas.remove(pointer);
  } else if (state.eagerWinner != null) {
    // 3.如果有eagerWinner，就让它接受，其他成员全部拒绝
    _resolveInFavorOf(pointer, state, state.eagerWinner!);
  }
  // 4.如果没有eagerWinner，且有多个成员，就等待后续处理
}
```
这里应该是符合第4种，等待后续处理的情况。

> 以上`down`事件流程，使用`GestureDetecotor`监听`onTap`点击事件，实际就是对`Listener`设置了`down`事件的监听，并让`TapGestureRecognizer`来初始化相关数据、注册到手势竞技场、注册`handleEvent`函数。然后到`GestureBinding`中，关闭竞技场，并等待后续处理。

3. 收到`up`事件
由于监听`onTap`，所以相应的`Listener`只注册了`down`事件的监听，那么`up`事件就可以直接看到`GestureBinding`的`handleEvent`函数：之前`down`事件流程执行`PrimaryPointerGestureRecognizer`中的`handleEvent`，实际没有任何作用，等待后续处理。而此时再次调用`pointerRouter.route(event)`来执行之前注册的`PrimaryPointerGestureRecognizer`中的`handleEvent`，执行`handlePrimaryPointer`，将有实际作用：
```
// BaseTapGestureRecognizer

void handlePrimaryPointer(PointerEvent event) {
  if (event is PointerUpEvent) {
    _up = event;
    _checkUp();
  } else if (event is PointerCancelEvent) {
    //...
  } else if (event.buttons != _down!.buttons) {
    //...
  }
}
```
再次看到`handlePrimaryPointer`，此时事件是`up`，所以会执行`_checkUp()`：
```
void _checkUp() {
  //...
  handleTapUp(down: _down!, up: _up!);
  _reset(); //最后重置
}
```
执行`handleTapUp`：
```
// TapGestureRecognizer

void handleTapUp({ required PointerDownEvent down, required PointerUpEvent up}) {
  //...
  switch (down.buttons) {
    case kPrimaryButton:
      //...
      if (onTap != null)
        invokeCallback<void>('onTap', onTap!);
      break;
    //...
    default:
  }
}
```
这里就会执行`onTap`函数。对于我们描述的父节点包含子节点，点击子节点的情况，由于子节点先注册了`handleEvent`函数，`PrimaryPointerGestureRecognizer`的`handleEvent`函数最后会判断是`up`或者`cancel`就会取消当前手指的注册函数，所以不会再调用父节点注册的`handleEvent`函数，也就解决了事件冲突。

而后还会执行`gestureArena`的`sweep`函数：
```
void sweep(int pointer) {
  final _GestureArena? state = _arenas[pointer];
  if (state == null)
    return; // This arena either never existed or has been resolved.
  if (state.isHeld) {
    state.hasPendingSweep = true;
    return; // This arena is being held for a long-lived member.
  }
  _arenas.remove(pointer); // 移除对应手指的手势竞技场
  if (state.members.isNotEmpty) {
    // First member wins.
    state.members.first.acceptGesture(pointer); //竞技成员的第一个为胜利者
    // Give all the other members the bad news.
    for (int i = 1; i < state.members.length; i++)
      state.members[i].rejectGesture(pointer);
  }
}
```
这里对于点击事件来说，实际就是收尾清除工作

## 总结
本文以点击事件为例，分析了事件分发和冲突处理的流程，但主要是顺着源码执行流程，如果要更深入来理解这套流程的设计思路，还需要再针对其他手势处理的源码来分析，以及结合实践。