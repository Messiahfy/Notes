## 1. Widget体系
* StatelessWidget：无状态的组合其他Widget的Widget
* StatefulWidget：有状态的组合其他Widget的Widget
* RenderObjectWidget：布局Widget
* ProxyWidget：代理Widget，例如InheritedWidget

## 2. 更新机制
StatefulElement的child Element就是其State构造的Widget对应的Element

1. if(child == null && newWidget != null)：根据Widget创建Element并mount
2. if(child == null && newWidget == null)：不做处理
3. if(child != null && newWidget == null)：deactiveChild，并detachRenderObject，添加Element到BuildOwner维护的_inactiveElementes中
4. if(child != null && newWidget != null)：如果child.widget和newWidget相等则不处理，否则根据Widget.canUpdate判断是否需要更新，是则调用child.update(newWidget)，否则调用deactiveChild和inflateWidget完成Element的替换。

## 3. 优化
1. setState范围尽可能小，避免没必要的高层次开始遍历。因为state会调用Element的performRebuild，重新构造Widget，并调用updateChild，其中调用child.update，如果child Element是有child的Element类型，例如SingleChildRenderObjectWidget，则还回递归调用updateChild。所以从更高的层次调用setState会导致更多的更新遍历。

markNeedsBuild-->BuildOwner.scheduleBuildFor（把当前Element添加到`_dirtyElements`）-->onBuildScheduled(WidgetsBinding._handleBuildScheduled)-->ensureVisualUpdate-->scheduleFrame-->_handleDrawFrame-->handleDrawFrame-->_handlePersistentFrameCallback-->WidgetsBinding.drawFrame-->BuildOwner.buildScope-->buildScope._flushDirtyElements，调用之前添加到`_dirtyElements`的element的rebuild


核心就是标记Element需要build，然后间接引发调用element的rebuild，`StatefulElement`、`StatefulElement`和`RenderObjectElement`等都是`ComponentElement`的子类，`ComponentElement`执行`performRebuild`将调用`build()`，从该层次的Element开始重新构建widget，并调用Element的updateChild，根据情况，如果是`RenderObjectElement`类型的`performRebuild`，还会触发`RenderObjectWidget.updateRenderObject`调用（一般内部会判断，设置相同值就忽略了，避免不必要的重绘），从而引发markNeedsLayout和markNeedsPaint和 RendererBinding.drawFrame引发布局绘制。

然后后续流程和Flutter应用启动中一致。

2. 会频繁重建，比如动画的情况，复杂widget可以作为成员变量或者其他方式缓存起来
3. RepaintBoundary：
4. RelayoutBoundary：例如constraints.isTight，也就是固定宽高，那么RenderObject的layout会把relayoutBoundary设置为true

> setState会引发当前Widget和子级Widget重建，不会导致父级Widget重建；但如果触发`RenderObjectElement`调用`markNeedsLayout`和`markNeedsPaint`的话，会向父级`RenderObject`调用传播（除非设置了relayoutBoundary），都标记为`_needsLayout`和`_needsPaint`，但在`layout`时还会考虑`sizedByParent`、`constraints.isTight`等条件，不需要布局则会跳过。绘制则会考虑`RepaintBoundary`，`markNeedsPaint()`方法内判断是`RepaintBoundary`则不会继续往父级传播。

总的来说：尽量避免rebuild widget，rebuild widget的情况下尽量符合更新逻辑中的相同判定，从而避免更新Element，

## 4. key
测试如下例子：
```
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
点击按钮，交换两个widget的位置，会发现并没有变化。因为`Wdiget`只是UI的配置，`Element`才是核心的UI元素，在前文介绍的`StatefulElement`更新机制中可以了解到，如果`Widget`使用equals判断相同，则不会做任何更新处理。刚好这里`_TestWidgetState`的`widgets`数组就是两个相同的`StatefulColor`，即使交换了对象的位置，`equals`判断还是返回true，所以不会更新。

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