&emsp;&emsp;Android用一个32位的整型值（int）表示一次`MotionEvent`事件。**低8位**表示事件的具体动作，比如按下，抬起，滑动，还有多点触控时的按下，抬起；**高8位**（更高的16位没有用）表示第几个触摸点。
* `getAction()`：返回信息包含具体动作和索引。在单点触控时和`getActionMasked()`的返回值相同，因为单点时高8位索引为0。

* `getActionMasked()`：只表示事件的具体动作，比如DOWN、MOVE。

* `getActionIndex()`：表示第几个触控点，例如：第一个触摸点则为0，同时有两个手指按在屏幕上时，则第二个触摸点返回1。

> 即`getAction()`返回的值包含了低8位的`getActionMasked()`和高8位的`getActionIndex()`（高16位没有用）。**例如**：`getAction()`返回值为**0x0000**，表示第一个触摸点（单点触控则为唯一的触摸点），动作为**DOWN**；若返回值为**0x0100**，表示第二个触摸点，动作为**DOWN**。

&emsp;&emsp;根据`getActionIndex()`返回的**索引**，可以传给`getPointerId(int pointerIndex)`方法，返回该触摸点的**标识ID**。
&emsp;&emsp;根据触摸点的**标识ID**也可以反过来得到索引：把**标识ID**传给`findPointerIndex (int pointerId)`方法，可以根据**标识ID**返回索引。
（估计第二个手指按下又抬起再按下，两次按下的index都是1，但是id应该不同，暂未测试）

&emsp;&emsp;多点触控时，第一个手指按下的事件为**ACTION_DOWN**，其他手指按下为**ACTION_POINTER_DOWN**。抬起时，第一个手指为**ACTION_UP**，其余手指为**ACTION_POINTER_UP**。而DOWN和UP之间的**ACTION_MOVE**事件是所有手指合在一个**MotionEvent**对象中。使用`getX (int pointerIndex)`之类的需要传入**pointerIndex**的方法可以区分不同的手指，无参`getX ()`返回第一个手指的数据。

## 事件位置
`MotionEvent.getX()`：获取点击事件**距离控件左边缘**的距离，单位：像素；
`MotionEvent.getY()`：获取点击事件**距离控件上边缘**的距离，单位：像素；
`MotionEvent.getRawX()`：获取点击事件**距离屏幕左边缘**的距离，单位：像素；
`MotionEvent.getRawY()`：获取点击事件**距离屏幕上边缘**的距离，单位：像素。