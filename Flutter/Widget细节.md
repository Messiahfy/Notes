## 1. Widget体系
* StatelessWidget：无状态的组合其他Widget的Widget
* StatefulWidget：有状态的组合其他Widget的Widget
* RenderObjectWidget：布局Widget
* ProxyWidget：代理Widget，例如InheritedWidget

## 2. 更新机制
StatefulElement的child Element就是其State构造的Widget对应的Element，可以查看`updateChild`方法源码

1. if(child == null && newWidget != null)：根据Widget创建Element并mount
2. if(child == null && newWidget == null)：不做处理
3. if(child != null && newWidget == null)：deactiveChild，并detachRenderObject，添加Element到BuildOwner维护的_inactiveElementes中
4. if(child != null && newWidget != null)：如果child.widget和newWidget相等则不处理，否则根据`Widget.canUpdate`判断是否需要更新，是则调用child.update(newWidget)，否则调用deactiveChild和inflateWidget完成Element的替换。

## 3. 优化
1. setState范围尽可能小，避免没必要的高层次开始遍历。因为state会调用Element的performRebuild，重新构造Widget，并调用updateChild，其中调用child.update，如果child Element是有child的Element类型，例如SingleChildRenderObjectWidget，则还回递归调用updateChild。所以从更高的层次调用setState会导致更多的更新遍历。

markNeedsBuild-->BuildOwner.scheduleBuildFor（把当前Element添加到`_dirtyElements`）-->onBuildScheduled(WidgetsBinding._handleBuildScheduled)-->ensureVisualUpdate-->scheduleFrame-->_handleDrawFrame-->handleDrawFrame-->_handlePersistentFrameCallback-->WidgetsBinding.drawFrame-->BuildOwner.buildScope-->buildScope._flushDirtyElements，调用之前添加到`_dirtyElements`的element的rebuild --> performRebuild()


核心就是标记Element需要build，然后间接引发调用element的rebuild，`StatefulElement`、`StatefulElement`和`RenderObjectElement`等都是`ComponentElement`的子类，`ComponentElement`执行`performRebuild`将调用`build()`，从该层次的Element开始重新构建widget，并调用Element的updateChild，根据情况，如果是`RenderObjectElement`类型的`performRebuild`，还会触发`RenderObjectWidget.updateRenderObject`调用（一般内部会判断，设置相同值就忽略了，避免不必要的重绘），从而引发markNeedsLayout和markNeedsPaint和 RendererBinding.drawFrame引发布局绘制。

然后后续流程和Flutter应用启动中一致。

2. 会频繁重建，比如动画的情况，复杂widget可以作为成员变量或者其他方式缓存起来
3. RepaintBoundary：
4. RelayoutBoundary：例如constraints.isTight，也就是固定宽高，那么RenderObject的layout会把relayoutBoundary设置为true

> setState会引发当前Widget和子级Widget重建，不会导致父级Widget重建；但如果触发`RenderObjectElement`调用`markNeedsLayout`和`markNeedsPaint`的话，会向父级`RenderObject`调用传播（除非设置了relayoutBoundary），都标记为`_needsLayout`和`_needsPaint`，但在`layout`时还会考虑`sizedByParent`、`constraints.isTight`等条件，不需要布局则会跳过。绘制则会考虑`RepaintBoundary`，`markNeedsPaint()`方法内判断是`RepaintBoundary`则不会继续往父级传播。

总的来说：尽量避免rebuild widget，rebuild widget的情况下尽量符合更新逻辑中的相同判定，从而避免更新Element，

## 4. key
测试如下例子：
```dart
class TestWidget extends StatefulWidget {
  const TestWidget({super.key});

  @override
  State<TestWidget> createState() => _TestWidgetState();
}

class _TestWidgetState extends State<TestWidget> {
  List<Widget> widgets = [
    StatefulColor(),
    StatefulColor(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Row(children: widgets,),
      floatingActionButton: FloatingActionButton(
        onPressed: switchWidget,
        child: Icon(Icons.undo),
      ),
    );
  }

  switchWidget() {
    widgets.insert(0, widgets.removeAt(1));
    setState(() {});
  }
}


class StatefulColor extends StatefulWidget {
  const StatefulColor({super.key});

  @override
  State<StatefulWidget> createState() => _StatefulColorState();
}

class _StatefulColorState extends State<StatefulColor> {
  final Color color = RandomColor().randomColor();

  @override
  Widget build(BuildContext context) {
    return Container(
      width: 100,
      height: 100,
      color: color,
    );
  }
}

class RandomColor {
  Color randomColor() {
    return Color((0xFF000000 + Random().nextInt(0x00FFFFFF)));
  }
}
```
点击按钮，交换两个widget的位置，会发现并没有变化。因为`Wdiget`只是UI的配置，`Element`才是核心的UI元素，在前文介绍的`StatefulElement`更新机制中可以了解到，如果`Widget`使用`Widget.canUpdate`判断相同（默认通过`runtimeType`和`key`判断），则不会做任何更新处理。刚好这里`_TestWidgetState`的`widgets`数组就是两个相同的`StatefulColor`，即使交换了对象的位置，`Widget.canUpdate`判断还是返回true，所以不会更新。

Flutter大多数时候不需要使用key，但如果增删或重新排序的情况就可能需要用到`key`，key用来让Element知道自己对应的Widget是哪个，比如两个equals的`StatefulWidget`子类对象换了位置，如果没有key区分，无法正确刷新Element的位置。

所以把`_TestWidgetState`的`widgets`数组改为如下代码即可正常刷新交换后的位置：
```
  List<Widget> widgets = [
    StatefulColor(key: UniqueKey()),
    StatefulColor(key: UniqueKey()),
  ];
```
* 抽象类`LocalKey`：
    * `ValueKey`：传入一个基本类型或者对象作为key，通过`==`判断是否相同
    * `ObjectKey`：传入一个基本类型或者对象作为key，必须是同一个地址的对象才认为相同。比如两个`Object()`不同，但两个`const Object()`相同；`2`和`1 + 1`相同。
    * `UniqueKey`：每次构造都是唯一的对象，不会相同。不过这样做，每次rebuild重新构造 UniqueKey 都会和之前不同，也就可能丢失状态，除非把构造它的位置提升到rebuild之外。
        * 子类`PageStorageKey`：用于保存和恢复页面状态的key。滚动视图、PageView等在滑动位置变化后，会将滚动位置自动保存到`PageStorage`中，保存的数据会和`PageStorageKey`关联。
* `GlobalKey`：`LocalKey`都是局部的，在当前层级找不到原本的key，就会重建新的Element。`Element.mount`的时候，会判断`widget.key`为`GlobalKey`类型时，将它保存到`BuildOwner`的`Map<GlobalKey, Element>`数据中，`GlobalKey`和它所在的`Element`关联，所以可以通过`GlobalKey`访问它对应的widget/element。并且即使层级变化，也能在rebuild时，根据`GlobalKey`找到原本关联的element复用，从而提升性能，也能保留Element中的状态（比如StatefulElement中的state）。可以查看`updateChild`源码，由于层级变化，widget不同，就会执行`inflateWidget`，如果是`GlobalKey`就会调用`_retakeInactiveElement`复用原本的element。
```
final GlobalKey _globalKey= GlobalKey();

@override
Widget build(BuildContext context) {
  return Row(
    children: [
      StatefulColor(key: _globalKey)
      // 假设rebuild时，变为 Center(child: StatefulColor(key: _globalKey))，可以保留状态
    ]
  );
}
```

## 5. StatefulWidget对应的State的生命周期
* `createState()`：创建 State 实例
* `initState()`：State 初始化状态
* `build()`：只负责“根据当前状态绘制 UI”，应该尽量保持纯函数、无副作用。它可能因为很多原因被频繁调用（父级 setState、布局变化、约束变化、依赖的 `InheritedWidget` 变化、滚动等）。
* `didChangeDependencies` 的作用，是在依赖的 `InheritedWidget`（如 Provider、Theme、MediaQuery、Localizations 等）发生变化时给你一个专用生命周期钩子，用来响应这些变化并执行必要的副作用（订阅、取消订阅、重建缓存、启动/停止动画等）。仅在依赖的 InheritedWidget变更时调用，以及在 initState 之后首次调用。它是“依赖变化”的专属钩子，适合执行一次性的副作用。**在状态管理的源码分析可知，触发 InheritedWidget update，就会调用依赖它的 Element 的`didChangeDependencies`方法**
* `didUpdateWidget`：widget的配置参数发生变化时回调，并且flutter framework之后必然会调用`build`。
* `activate()`：
* `deactivate()`：当 State 对象暂时从 Widget 树中移除时调用。可能在 dispose 前调用，也可能重新插入到树中
* `dispose()`：当 State 对象被永久从 Widget 树中移除时调用。可用于释放资源（取消订阅、关闭流、销毁定时器）、

在第三节的**优化**部分，可知 StatefulWidget 的 State调用`setState`后，触发 StatefulElement 的 performRebuild() 方法调用，根据情况调用 `didChangeDependencies` ，然后调用 build 构造新的 widget，传入 updateChild 方法，执行更新，如果单纯更新widget的参数配置，没有更换其他类型的widget或者删除，则element对象还是原本的，只是会更新widget属性，以及调用一些生命周期，比如 `didUpdateWidget`。否则会创建新的element并mount。

> `build()`、`didChangeDependencies()`和`didUpdateWidget()`都是在更新的时候触发，但场景有所不同。`build()`用于在任何需要重建 UI 时渲染 UI，不能包含副作用；`didChangeDependencies()`用于依赖的 InheritedWidget 变化时	响应依赖变化，执行副作用操作。`didUpdateWidget()`用于处理同一类型Widget的属性变化（即Widget重建但State被复用的场景），当父Widget重新构建并创建了 新的Widget实例 （但 `key` 和 `runtimeType` 与旧Widget相同）时，State会被复用，此时 `initState()` 不会调用，但 `didUpdateWidget()` 会被触发，可以同步新旧状态。

有 `didChangeDependencies` 就一定有 `build`，但反之不是，build 可能因为其他原因被调用


StatefulElement被`InheritedWidget`调用 didChangeDependencies 后，会设置 `_didChangeDependencies = true`，然后执行`performRebuild`，其中判断 _didChangeDependencies 为 true， 就会调用 State 的 `didChangeDependencies`，并且 rebuild。

比如 build 可能因为父组件重建而被频繁调用
    // 但昂贵的资源更新只在 didUpdateWidget 中处理
如果 StatefulWidget 中的数据全部依赖viewModel（ChangeNotifier）、Provider，就还好。但如果widget/state中自己存了一些状态、数据，则需要考虑使用`didChangeDependencies()`和`didUpdateWidget()`来处理

**StatefulElement复用时，State 对象也会复用**，此时 `initState()`只调一次，所以有些情况也需要`didChangeDependencies()`和`didUpdateWidget()`，而不能只靠`initState()`，否则缓存在State对象中的状态可能被错误复用，导致状态错乱。


使用建议
- 在 didChangeDependencies 中读取依赖（如 context.read），比较旧实例和新实例，只有发生变化时才重订阅。
- 在 dispose 中取消订阅，保持资源清理。
- 保持 build 纯粹，只渲染 UI，不做订阅/取消订阅/网络请求/持久化等副作用。

## 6. slot
首先看看源码里面，slot 在哪里创建和使用
```dart
abstract class RenderObjectElement extends Element {

  // mount 可以传入 slot
  @override
  void mount(Element? parent, Object? newSlot) {
    super.mount(parent, newSlot);
    _renderObject = (widget as RenderObjectWidget).createRenderObject(this);
    attachRenderObject(newSlot);
    super.performRebuild();
  }

  @override
  void attachRenderObject(Object? newSlot) {
    _slot = newSlot;
    _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
    // 插入到祖先 RenderObjectElement 时，会传入slot，
    _ancestorRenderObjectElement?.insertRenderObjectChild(renderObject, newSlot);
    final List<ParentDataElement<ParentData>> parentDataElements =
        _findAncestorParentDataElements();
    for (final ParentDataElement<ParentData> parentDataElement in parentDataElements) {
      // 对于自带 ParentData 的类型，执行初始化，比如 Stack 中的 Positioned 类型
      _updateParentData(parentDataElement.widget as ParentDataWidget<ParentData>);
    }
  }

}

// 单个 child 的情况，slot没什么用，所以传入null
class SingleChildRenderObjectElement extends RenderObjectElement {

  Element? _child;

  @override
  void mount(Element? parent, Object? newSlot) {
    super.mount(parent, newSlot);
    _child = updateChild(_child, (widget as SingleChildRenderObjectWidget).child, null);
  }

  // ...
}

// 多个 child 的情况，会创建 IndexedSlot，包含了index和previousChild信息，RenderObject中可以通过它形成链表
class MultiChildRenderObjectElement extends RenderObjectElement {

  @override
  void mount(Element? parent, Object? newSlot) {
    super.mount(parent, newSlot);
    final MultiChildRenderObjectWidget multiChildRenderObjectWidget =
        widget as MultiChildRenderObjectWidget;
    final List<Element> children = List<Element>.filled(
      multiChildRenderObjectWidget.children.length,
      _NullElement.instance,
    );
    Element? previousChild;
    for (int i = 0; i < children.length; i += 1) {
      // 为每个 child 创建 IndexedSlot
      final Element newChild = inflateWidget(
        multiChildRenderObjectWidget.children[i],
        IndexedSlot<Element?>(i, previousChild),
      );
      children[i] = newChild;
      previousChild = newChild;
    }
    _children = children;
  }

  @override
  void insertRenderObjectChild(RenderObject child, IndexedSlot<Element?> slot) {
    final ContainerRenderObjectMixin<RenderObject, ContainerParentDataMixin<RenderObject>>
    renderObject = this.renderObject;
    // slot 用于在 renderObject 中插入子级时，传递位置信息
    renderObject.insert(child, after: slot.value?.renderObject);
  }

  @override
  void update(MultiChildRenderObjectWidget newWidget) {
    super.update(newWidget);
    final MultiChildRenderObjectWidget multiChildRenderObjectWidget =
        widget as MultiChildRenderObjectWidget;

    // updateChildren 内会构造 slot，传入 updateChild 方法中
    _children = updateChildren(
      _children, // 旧的子级列表 List<Element>
      multiChildRenderObjectWidget.children, // 新的 widget
      forgottenChildren: _forgottenChildren,
    );
    _forgottenChildren.clear();
  }

  // ...
}
```
单个子级的`SingleChildRenderObjectElement`不需要slot，因为不需要关心顺序关系。而`MultiChildRenderObjectElement`包含多个子级，所以需要`slot`信息将兄弟节点按顺序形成链表排列起来。

`MultiChildRenderObjectWidget`（`Row`、`Column`、`Stack`等都是）对应的 `MultiChildRenderObjectElement` 的 mount 方法可以看到给子级使用的是 `IndexedSlot`。slot 也可以是任意的自定义数据结构，便于支持任意布局，比如表格`Table` slot 是 (row, col)

然后我们看看`renderObject.insert`怎么使用 slot：
```dart
mixin ContainerRenderObjectMixin<ChildType extends RenderObject, ParentDataType extends ContainerParentDataMixin<ChildType>> on RenderObject {

  // 这个 after 就是 slot 中存储的前一个 renderObject
  void insert(ChildType child, { ChildType? after }) {
    adoptChild(child);
    _insertIntoChildList(child, after: after);
  }

  void adoptChild(RenderObject child) {
    setupParentData(child); // 初始化 ParentData
    markNeedsLayout();
    markNeedsCompositingBitsUpdate();
    markNeedsSemanticsUpdate();
    child._parent = this;
    if (attached) {
      child.attach(_owner!);
    }
    redepthChild(child);
  }

  // 形成链表
  void _insertIntoChildList(ChildType child, { ChildType? after }) {
    final ParentDataType childParentData = child.parentData! as ParentDataType;
    _childCount += 1;
    if (after == null) {
      childParentData.nextSibling = _firstChild;
      if (_firstChild != null) {
        final ParentDataType firstChildParentData = _firstChild!.parentData! as ParentDataType;
        firstChildParentData.previousSibling = child;
      }
      _firstChild = child;
      _lastChild ??= child;
    } else {
      final ParentDataType afterParentData = after.parentData! as ParentDataType;
      if (afterParentData.nextSibling == null) {
        childParentData.previousSibling = after;
        afterParentData.nextSibling = child;
        _lastChild = child;
      } else {
        childParentData.nextSibling = afterParentData.nextSibling;
        childParentData.previousSibling = after;
        final ParentDataType childPreviousSiblingParentData = childParentData.previousSibling!.parentData! as ParentDataType;
        final ParentDataType childNextSiblingParentData = childParentData.nextSibling!.parentData! as ParentDataType;
        childPreviousSiblingParentData.nextSibling = child;
        childNextSiblingParentData.previousSibling = child;
      }
    }
  }

}
```
`Element` 存储了 slot，但 `RenderObject` 并不存储，只是通过 `slot` 配置 `ParentData` 相关信息，形成例如链表顺序结构。

源码中，slot的创建和传递流程是这样的：
```
rebuild --> performRebuild --> updateChild --> update --> updateChildren（MultiChildRenderObjectWidget情况）--> 生成slot --> 递归 updateChild/updateChildren
```
在`updateChildren`中，根据类型和key判断 element 是否可以复用，对于 `Element` 来说，slot 重点还是在`updateChild`中对于复用的 element，判断 slot 是否相同，不相同则需要更新，触发 RenderObject（ContainerRenderObjectMixin）的`move`方法，调整链表顺序，重新布局绘制。

`MultiChildRenderObjectElement`的情况，`updateChildren`生成的slot就是当前的位置信息，所以对于复用的element，位置变化就会导致slot变化，RenderObject再根据slot的信息，修改链表顺序情况。

> `_TableElement`则是在`update`方法中就构造了`slot`，再传入`updateChildren`中

`ContainerRenderObjectMixin`（用于RenderObject）中使用链表结构管理子级 `RenderObject`，slot（IndexedSlot）就存储了index和前一个renderObject ，根据 slot 的信息，renderObject的 parentData 中可以记录相关信息（RenderObject 不存储 slot，只用它做一次性的插入/移动决策，最终信息形成在 parentData 中），从而形成链表。相比直接存储 `List<RenderObject>` ，链表操作时间复杂度为 O(1)

* RenderObject 需要知道 child 顺序，比如 Stack 就是根据 slot 构造了 child 链表，从而形成正确的绘制顺序
* 查看各种 Element 的 mount 方法，有多个子级的类型，就会自定义 slot，比如 MultiChildRenderObjectElement、_TableElement（Table）
* RenderObject adoptChild 调用 setupParentData(child) 会初始化一个 ParentData（如果没有的话，因为ParentDataWidget类型会自带，在 attachRenderObject 中就创建了，比如Stack的Positioned），各个类型都可以自定义
* parentData有很多子类，用于各种场景的定制信息，可以查看 parentData 的继承关系结构

### 为什么需要 slot？
#### 1. 性能优化
* slot 让父 Element 能 `O(1)` 知道子 Element 的当前位置，无需每次遍历或构建映射。

#### 2. 解耦位置与身份
* Key = “你是谁”
* Slot = “你现在在哪”
两者分离，设计更清晰
支持非线性布局

比如一个父容器有多个命名插槽：
```dart
MyLayout(
  header: Widget(key: 'h'),
  body: Widget(key: 'b'),
)
```
这里 slot 可以是 'header' 和 'body'，而不是数字索引。

## 7. 页面状态保持

### KeepAlive
如果你的页面 Widget 混入 `AutomaticKeepAliveClientMixin`：
* 并返回 `wantKeepAlive` = true
* ListView等组件 会跳过 dispose，保留 Element 和 State

> 底层：KeepAlive 机制通过 `KeepAliveParentDataMixin` 和 `SliverMultiBoxAdaptorElement` 协作，标记该 child 需要“存活”。

可以调试看SliverList的层级，包含了 `AutomaticKeepAlive`

```dart
mixin AutomaticKeepAliveClientMixin<T extends StatefulWidget> on State<T> {
  KeepAliveHandle? _keepAliveHandle;

  // 核心方法
  void _ensureKeepAlive() {
    _keepAliveHandle = KeepAliveHandle();
    // 发送通知
    KeepAliveNotification(_keepAliveHandle!).dispatch(context);
  }

  void _releaseKeepAlive() {
    _keepAliveHandle!.dispose();
    _keepAliveHandle = null;
  }

  @protected
  bool get wantKeepAlive; // 为true就会调用 _ensureKeepAlive()

  @protected
  void updateKeepAlive() {
    if (wantKeepAlive) {
      if (_keepAliveHandle == null) {
        _ensureKeepAlive();
      }
    } else {
      if (_keepAliveHandle != null) {
        _releaseKeepAlive();
      }
    }
  }

  @override
  void initState() {
    super.initState();
    if (wantKeepAlive) {
      _ensureKeepAlive();
    }
  }

  @override
  void deactivate() {
    if (_keepAliveHandle != null) {
      _releaseKeepAlive();
    }
    super.deactivate();
  }

  @mustCallSuper
  @override
  Widget build(BuildContext context) {
    if (wantKeepAlive && _keepAliveHandle == null) {
      _ensureKeepAlive();
    }
    return const _NullWidget(); // 返回的 Widget 并不会被使用，只是要继承 State 的方法占位
  }
}
```
* `wantKeepAlive`为true，就会触发`_ensureKeepAlive()`调用，将 `KeepAliveNotification` 通知发出去。
* `AutomaticKeepAlive` 是一个 StatefulWidget，它内部添加了`NotificationListener`用于监听`KeepAliveNotification`，监听到之后就会给包裹实际 child 的`KeepAlive`更新`ParentData`数据，设置其中的`keepAlive`值。
* `RenderSliverMultiBoxAdaptor`（ListView等组件内部会使用它）在需要回收item时就会考虑这个`keepAlive`值，从而决定缓存还是删除。

### PageStorageKey
比 `AutomaticKeepAliveClientMixin` 更轻量（不 keep alive 整个 widget），只保存关键数据。

使用核心流程：
1. 创建一个 `PageStorageBucket`，`PageStorage`持有它，并包裹 child widget
2. 在需要保存状态的子 widget 中使用 `PageStorageKey`
3. 使用`PageStorage.of(context)`调用 `readState` 和 `writeState` 来读写状态

> 本质上就是状态提升，并存到一个 Map 结构中

使用示例：
```dart
class PageStorageDemo extends StatefulWidget {
  @override
  _PageStorageDemoState createState() => _PageStorageDemoState();
}

class _PageStorageDemoState extends State<PageStorageDemo> {
  final PageController _controller = PageController();
  final PageStorageBucket _bucket = PageStorageBucket();
  final List<Widget> _pages = [];

  @override
  void initState() {
    super.initState();
    // child 持有 PageStorageKey
    _pages.addAll([
      _PageWithStorage(key: PageStorageKey('page1'), color: Colors.red),
      _PageWithStorage(key: PageStorageKey('page2'), color: Colors.green),
      _PageWithStorage(key: PageStorageKey('page3'), color: Colors.blue),
    ]);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('PageStorage 状态保持')),
      // 构造 PageStorage ，传入 PageStorageBucket
      body: PageStorage(
        bucket: _bucket,
        child: PageView(
          controller: _controller,
          children: _pages,
        ),
      ),
    );
  }
}

class _PageWithStorage extends StatefulWidget {
  final Color color;

  const _PageWithStorage({
    Key? key,
    required this.color,
  }) : super(key: key);

  @override
  __PageWithStorageState createState() => __PageWithStorageState();
}

class __PageWithStorageState extends State<_PageWithStorage> {
  final ScrollController _scrollController = ScrollController();
  List<String> _items = [];

  @override
  void initState() {
    super.initState();
    _items = List.generate(50, (index) => '项目 $index');
    
    // 恢复滚动位置
    final offset = PageStorage.of(context)?.readState(context) as double?;
    if (offset != null) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        _scrollController.jumpTo(offset);
      });
    }
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    // 保存滚动位置
    PageStorage.of(context)?.writeState(context, _scrollController.offset);

    return ListView.builder(
      controller: _scrollController,
      itemCount: _items.length,
      itemBuilder: (context, index) {
        return Container(
          margin: EdgeInsets.all(4),
          padding: EdgeInsets.all(16),
          color: widget.color.withOpacity(0.2),
          child: Text(_items[index]),
        );
      },
    );
  }
}
```

## 8. FocusNode、Shortcuts + Actions
`FocusNode`、`Shortcuts` 和 `Actions` 是构建键盘交互、快捷键和焦点管理的核心机制，它们共同构成了 Flutter 的响应式输入系统。
* `FocusNode`：管理 widget 的焦点状态（谁可以接收键盘输入）
* `Shortcuts widget`：监听键盘事件，匹配快捷键（如 Ctrl+C）。需要我们注册`ShortcutActivator`，用于定义触发快捷键的条件（比如`SingleActivator(LogicalKeyboardKey.keyC, control: true)` 表示 Ctrl+C 组合键），和对应的`Intent`(比如`DismissIntent`表示关闭当前焦点widget的意图)，Intent 对应的实际操作是 `Action`，在`Actions`中注册。
* `Actions widget`：定义快捷键的行为（如“复制”操作），和 `Shortcuts` 配合使用。

`Shortcuts` 是一个 StatefulWidget，它内部会创建 `Focus`（但设置为不能获取焦点，为了子级焦点不处理事件时，冒泡到这里处理），`Focus`也是一个 StatefulWidget，没有传入 FocusNode 时，`Focus` 会自动创建一个`FocusNode`。按键事件发生时，子级如果没有处理，就会冒泡到`Shortcuts`中处理，根据`Actions`中注册的`Intent`和`Action`对应关系找到匹配的`Action`来执行。比如`_DismissModalAction`用于关闭当前焦点页面：
```dart
class _DismissModalAction extends DismissAction {
  _DismissModalAction(this.context);

  final BuildContext context;

  @override
  bool isEnabled(DismissIntent intent) {
    final ModalRoute<dynamic> route = ModalRoute.of<dynamic>(context)!;
    return route.barrierDismissible;
  }

  @override
  Object invoke(DismissIntent intent) {
    // 关闭当前页面
    return Navigator.of(context).maybePop();
  }
}
```

> `Shortcuts`接收到冒泡事件，会以接收事件的组件的`context`开始查找组件树父级中的`Actions`，从而找到对应的`Action`。所以`Actions`作为`Shortcuts`的子级，依然可以生效，只要`Actions`是接收到事件的焦点组件的父级即可。一般情况，快捷键定义是统一的，实际的行为可能根据UI区域有不同实现，所以一般`Shortcuts`为`Actions`父级；但如果执行行为是统一的，而快捷键不同，则可以考虑`Actions`为`Shortcuts`的父级。

> `ShortcutRegistrar`用于动态、临时的快捷键需求（内部也会使用`Shortcuts`），`Shortcuts + Actions`用于静态的快捷键配置需求。


* `Focus` widget：管理一个普通的`FocusNode`，用于单个可聚焦元素（如按钮、输入框）。
* `FocusScope` widget：是 `Focus` 的子类，管理一个 `FocusScopeNode`，`FocusScopeNode`是`FocusNode`的子类。通过 `FocusScope.of(context)`可以获得`FocusScopeNode`根结点。

### 事件传播路径：从焦点节点到根节点
当系统接收到按键事件时，FocusManager会按以下顺序传播事件：
1. 步骤1：早期处理器（Early Handlers） 先调用通过addEarlyKeyEventHandler注册的全局早期处理器。如果任何处理器返回KeyEventResult.handled或skipRemainingHandlers，事件传播直接终止。
2. 步骤2：焦点树传播（核心流程） 从`primaryFocus`开始，沿焦点树向上遍历其所有祖先节点，依次调用每个节点的`onKeyEvent`回调。
  * 每个`FocusNode`可以通过`onKeyEvent`决定是否处理事件：
    * 返回KeyEventResult.handled：事件被处理，传播终止。
    * 返回KeyEventResult.skipRemainingHandlers：事件未被处理，但停止向后续节点传播。
    * 返回KeyEventResult.ignored：事件未被处理，继续向上传播。
3. 步骤3：晚期处理器（Late Handlers） 如果焦点树中没有节点处理事件，再调用通过addLateKeyEventHandler注册的全局晚期处理器，逻辑同上。