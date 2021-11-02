### MediaQuery和软键盘
WidgetsApp（MaterialApp或者CupertinoApp的内部都会用它）内会使用MediaQuery来包裹child，所以可以用MediaQuery.of来获取数据。Scaffold内部会根据resize等属性，把根据MediaQuery得到的MediaQueryData修改后，再把child包装于Scaffold内的MediaQuery中，并且此时child访问的MediaQuery就是Scaffold内的MediaQuery。

WidgetsApp内的MediaQuery位于_MediaQueryFromWindow中，_MediaQueryFromWindow会注册WidgetsBinding的监听，例如软键盘弹起，会回调didChangeMetrics，此时调用setState重新build，会创建新的MediaQuery及更新后的MediaQueryData。由于MediaQuery是InheritedWidget类型，Scaffold内依赖了它，所以发生变化时，会通知Scaffold重建改变高度等属性。

### Theme
无论是MaterialApp还是CupertinoApp，都可以传入Theme，MaterialApp中是ThemeData类型，CupertinoApp中是CupertinoThemeData类型，不设置的话就都会使用默认的主题。使用MaterialApp和CupertinoApp时，我们的widget都会被直接或间接包裹于theme相关的InheritedWidget类型的Widget中，所以可以通过Theme.of(context)或者CupertinoTheme.of(context)来获取到主题对象。

### Container
使用Container时如果没有使用修饰相关的参数，仅传入child，那么就是简单的直接嵌套child，没有其他间接嵌套。

### Material
名为Material的Widget，在很多Material风格的Widget中使用，用Material来包裹其他Widget，可以提供裁剪、Z轴阴影高度、Ink effects（按钮水波纹等效果）。

### InkWell
Cupertino风格的Button，就是常规的Widget嵌套封装而成，而Material风格的Button由于其略复杂的设计风格，以及考虑焦点、鼠标的情况，所以设计有所不同。

Material风格的按钮基本都是使用RawMaterialButton，RawMaterialButton内部就是使用Material包裹InkWell，InkWell继承于InkResponse，InkResponse内部嵌套了Focus、MouseRegion、GestureDetector来负责处理焦点、鼠标、触摸事件。可以看出，Material风格的按钮，核心就是Material来提供裁剪、阴影，InkResponse处理交互事件，并配合完成Ink effects。


InkResponse对应的InkResponseState的build方法：
```
Widget build(BuildContext context) {
    super.build(context); // See AutomaticKeepAliveClientMixin.
    for (final _HighlightType type in _highlights.keys) {
      _highlights[type]?.color = getHighlightColorForType(type);
    }

    const Set<MaterialState> pressed = <MaterialState>{MaterialState.pressed};
    _currentSplash?.color = widget.overlayColor?.resolve(pressed) ?? widget.splashColor ?? Theme.of(context).splashColor;

    final MouseCursor effectiveMouseCursor = MaterialStateProperty.resolveAs<MouseCursor>(
      widget.mouseCursor ?? MaterialStateMouseCursor.clickable,
      <MaterialState>{
        if (!enabled) MaterialState.disabled,
        if (_hovering && enabled) MaterialState.hovered,
        if (_hasFocus) MaterialState.focused,
      },
    );
    return _ParentInkResponseProvider(
      state: this,
      child: Actions(
        actions: _actionMap,
        child: Focus(
          focusNode: widget.focusNode,
          canRequestFocus: _canRequestFocus,
          onFocusChange: _handleFocusUpdate,
          autofocus: widget.autofocus,
          child: MouseRegion(
            cursor: effectiveMouseCursor,
            onEnter: _handleMouseEnter,
            onExit: _handleMouseExit,
            child: Semantics(
              onTap: widget.excludeFromSemantics || widget.onTap == null ? null : _simulateTap,
              onLongPress: widget.excludeFromSemantics || widget.onLongPress == null ? null : _simulateLongPress,
              child: GestureDetector(
                onTapDown: enabled ? _handleTapDown : null,
                onTap: enabled ? _handleTap : null,
                onTapCancel: enabled ? _handleTapCancel : null,
                onDoubleTap: widget.onDoubleTap != null ? _handleDoubleTap : null,
                onLongPress: widget.onLongPress != null ? _handleLongPress : null,
                behavior: HitTestBehavior.opaque,
                excludeFromSemantics: true,
                child: widget.child,
              ),
            ),
          ),
        ),
      ),
    );
  }
```
整体设计比较清晰，但如何完成Ink effects还不清楚，这里以_handleTapDown和_handleTap为切入点来分析。_handleTapDown内会调用_startSplash：
```
void _startSplash({TapDownDetails? details, BuildContext? context}) {
  final Offset globalPosition;
  if (context != null) {
    final RenderBox referenceBox = context.findRenderObject()! as RenderBox;
    assert(referenceBox.hasSize, 'InkResponse must be done with layout before starting a splash.');
    globalPosition = referenceBox.localToGlobal(referenceBox.paintBounds.center);
  } else {
    globalPosition = details!.globalPosition;
  }
  final InteractiveInkFeature splash = _createInkFeature(globalPosition);//创建InteractiveInkFeature，例如InkRipple
  _splashes ??= HashSet<InteractiveInkFeature>();
  _splashes!.add(splash);
  _currentSplash = splash;
  updateKeepAlive();
  updateHighlight(_HighlightType.pressed, value: true);//创建InkHighlight（也是InteractiveInkFeature）
}
```
创建任何InteractiveInkFeature，都会直接开始执行动画。所以手指按下的时候，就会开始创建并执行水波纹之类的动画（根据设置而定）。而点击结束的_handleTap()就是执行动画结束等操作。
```
void _handleTap() {
  _currentSplash?.confirm();
  _currentSplash = null;
  updateHighlight(_HighlightType.pressed, value: false);
  if (widget.onTap != null) {
    if (widget.enableFeedback)
      Feedback.forTap(context);
    widget.onTap?.call();
  }
}
```
