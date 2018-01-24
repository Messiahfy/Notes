* [setContentView()](#setContentView())

## setContentView()  
setContentView()就是在DecorView（Activity的根视图，其根布局为垂直Linearlayout）的ContentView(Framelayout)中添加View或ViewGroup

## 启动Activity以返回结果
Activity A中调用调用startActivityForResult()，且实现onActivityResult()回调方法。  
Activity B中调用setResult()。

## 结束Activity
1. 通过调用 Activity 的 finish() 方法来结束该 Activity。  
2. 还可以通过调用 finishActivity() 结束之前通过startActivityForResult()启动的另一个 Activity。（疑问：已经跳转到activity B，若B没有销毁，则不会回调A的onActivityResult()，那么如何在activity A中执行finishActivity()。回答：目前测试可以在A中设定定时器，定时调用finishActivity()，其他方式和用途暂不了解）。

## 生命周期
![官方Activity生命周期图](https://developer.android.google.cn/images/activity_lifecycle.png)
* onStart()：即将出现在前台
* onResume()：已经在前台，即将开始与用户交互
* onPause()：失去焦点，但仍然部分可见
* onStop()：不可见
