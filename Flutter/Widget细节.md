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

markNeedsBuild-->scheduleBuildFor（把当前Element添加到`_dirtyElements`）-->onBuildScheduled(WidgetsBinding._handleBuildScheduled)-->ensureVisualUpdate-->scheduleFrame-->_handleDrawFrame-->handleDrawFrame-->_handlePersistentFrameCallback-->WidgetsBinding.drawFrame-->buildScope(调用之前添加到`_dirtyElements`的element的rebuild，如果调用RenderObjectElement类型的update可能会间接引发markNeedsLayout和markNeedsPaint) 和 RendererBinding.drawFrame引发布局绘制。

核心就是标记Element需要build，然后间接引发调用element的rebuild，从该层次的Element开始重新构建widget，并调用Element的updateChild，根据情况，可能标记需要布局绘制，最后调用RendererBinding.drawFrame引发布局绘制。

然后后续流程和Flutter应用启动中一致。

2. 会频繁重建，比如动画的情况，复杂widget可以作为成员变量或者其他方式缓存起来
3. RepaintBoundary：
4. RelayoutBoundary：例如constraints.isTight，也就是固定宽高，那么RenderObject的layout会把relayoutBoundary设置为true

## 4. key
GlobalKey可以访问其他widget的state，从而访问其方法或属性
