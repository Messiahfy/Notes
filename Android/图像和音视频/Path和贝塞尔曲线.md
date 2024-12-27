canvas要绘制任意路径的图形，就要使用到Path，还可以用用于剪切或者绘制文字路径。
基本操作参考[Path之基本操作1](http://www.gcssloop.com/customview/Path_Basic/)和[Path之基本操作2](http://www.gcssloop.com/customview/Path_Over)

贝塞尔曲线参考[Path之贝塞尔曲线](http://www.gcssloop.com/customview/Path_Bezier)


起始点默认原点，可用path.moveTo移动
二阶曲线：path.quadTo(control.x, control.y, end.x, end.y);
三阶曲线：path.cubicTo(control1.x, control1.y, control2.x,control2.y, end.x, end.y);


`PathMeasure`可用于对`Path`判断是否闭合、获取长度、截取片段，获取指定位置的坐标和切线值或者matrix