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

markNeedsBuild-->BuildOwner.scheduleBuildFor（把当前Element添加到`_dirtyElements`）-->onBuildScheduled(WidgetsBinding._handleBuildScheduled)-->ensureVisualUpdate-->scheduleFrame-->_handleDrawFrame-->handleDrawFrame-->_handlePersistentFrameCallback-->WidgetsBinding.drawFrame-->BuildOwner.buildScope，调用之前添加到`_dirtyElements`的element的rebuild，


核心就是标记Element需要build，然后间接引发调用element的rebuild，`StatefulElement`、`StatefulElement`和`RenderObjectElement`等都是`ComponentElement`的子类，`ComponentElement`执行`performRebuild`将调用`build()`，从该层次的Element开始重新构建widget，并调用Element的updateChild，根据情况，如果是`RenderObjectElement`类型的`performRebuild`，还会触发`RenderObjectWidget.updateRenderObject`调用，从而引发markNeedsLayout和markNeedsPaint和 RendererBinding.drawFrame引发布局绘制。

然后后续流程和Flutter应用启动中一致。

2. 会频繁重建，比如动画的情况，复杂widget可以作为成员变量或者其他方式缓存起来
3. RepaintBoundary：
4. RelayoutBoundary：例如constraints.isTight，也就是固定宽高，那么RenderObject的layout会把relayoutBoundary设置为true

> setState会引发当前Widget和子级Widget重建，不会导致父级Widget重建；但如果触发`RenderObjectElement`调用`markNeedsLayout`和`markNeedsPaint`的话，会向父级`RenderObject`调用传播（除非设置了relayoutBoundary），都标记为`_needsLayout`和`_needsPaint`，但在`layout`时还会考虑`sizedByParent`、`constraints.isTight`等条件，不需要布局则会跳过。绘制则会考虑`RepaintBoundary`，`markNeedsPaint()`方法内判断是`RepaintBoundary`则不会继续往父级传播。

## 4. key
GlobalKey可以访问其他widget的state，从而访问其方法或属性
