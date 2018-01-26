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

## Activity跳转的生命周期
1. Activity A 的onPause()方法执行。
2. Activity B 的 onCreate(), onStart(), 和 onResume() 依次执行. (Activity B 现在获得用户焦点。)
3. 然后，如何Activity A 在屏幕上不再可见, 它的 onStop() 方法执行。

## 异常生命周期及保存和恢复Activity状态
### 异常生命周期情况
1. 资源相关的系统配置发生改变导致Activity被杀死并重新创建（如横竖旋转屏幕）
2. 资源内存不足导致低优先级的Activity被杀死
### 保存和恢复Activity状态
* 异常情况下，可通过onSaveInstanceState()存储数据，通过onCreate()或者onRestoreInstanceState()恢复数据（onCreate需要判空）。只有在异常情况下，系统才会调用onSaveInstanceState()和onRestoreInstanceState()，正常的生命周期不会（如用户按了返回键或者调用了finish()方法）。  
* 在情况1下，可以通过设定来禁止屏幕旋转等。  
* 默认情况下，在onSaveInstanceState()和onRestoreInstanceState()方法中，系统自动为我们做了一定的恢复工作。当Activity在异常情况下需要重新创建时，系统会默认自动为我们保存当前Activity的视图结构，并且在Activity重启的时候恢复这些数据。系统统使用Bundle实例状态来保存活动布局中每个View对象的信息（如输入到EditText小部件中的文本值）。具体针对某一个特定的View系统能为我们恢复哪些数据，可以查看View源码，和Activity一样，每个View都有onSaveInstanceState()和onRestoreInstanceState()方法。
* 关于保存和恢复View层次结构，系统的工作流程：Activity调用onSaveInstanceState()，然后Activity会委托Window去保存数据，接着Window委托顶级容器（DecorView）去保存数据，DecorView再一一通知子元素保存数据。onRestoreInstanceState()的过程类似。
* 虽然系统会自动保存和恢复一些数据，但某些数据仍需自行保存和恢复。Bundle不适合保存过多的数据，因为消耗内存。过多的数据可以考虑采取综合的方式，持久化存储、onSaveInstanceState()和ViewModel等。

## 总结活动和任务的默认行为
* 当 Activity A 启动 Activity B，Activity A停止，但系统保存A的状态（如滚动位置和输入到表单中的文本）。如果用户在活动B中按下后退按钮，则活动A将恢复其状态。
* 当用户通过按Home按钮离开一个任务时，当前活动停止并且其任务进入后台。系统保留任务中每个活动的状态。如果用户稍后通过选择启动该任务的启动器图标来恢复该任务，则该任务进入前台并重新开始任务栈顶部的活动。
* 如果用户按下“后退”按钮，则当前活动从任务栈中弹出并销毁。 任务栈中的前一个活动恢复。当一个活动被销毁时，系统不保留活动的状态。
* 活动可以实例化多次，甚至可以从其他任务。

## 管理Activity的启动模式和任务栈
>   如果活动A启动活动B，则活动B可以在其清单文件中定义如何与当前任务相关联（如果有的话），活动A也可以通过Intent的flag请求活动B如何与当前任务相关联。 如果两个活动都定义活动B如何与任务相关联，则活动A的请求（如意图中定义的）覆盖活动B的请求（如其清单中所定义的）。  
**注意**：清单文件可用的某些启动模式不可用作意图标志，同样，某些意图标志可用的启动模式也不能在清单中定义。

### 使用manifest文件
> 在清单文件的<activity>的launchMode属性中指定  
1. standard 谁创建了该Activity，该Activity就运行于该任务栈中。Activity可以多次实例化，一个任务栈可以有多个实例，每个实例可以属于不同的任务栈。
2. singleTop 如果当前Activity已经存在于当前任务栈（当前任务栈意思是创建该activity的任务栈）的栈顶，则系统通过调用onNewIntent()将Intent传给该Activity实例，而不会创建该Activity的新实例。如果不位于栈顶，则会创建新实例。
3. singelTask 如果没有设定taskAffinity，则activity将属于应用默认的任务栈中；如果设定了taskAffinity，则activity将属于taskAffinity同名的任务栈中。可以看出singleTask表示activity只会存在一个实例，且只属于一个任务栈。
  * 当activity的实例不存在，则创建activity的实例于其应该属于的任务栈中。
  * 当activity的实例已存在，则不会创建新实例，调用其onNewIntent()方法，并且将该实例之上的activity实例全部出栈。
4. singelInstance 启动一个新的任务栈，并将此Activity作为该栈的唯一成员。如果该Activity已存在，则不会创建该Activity的新实例，而是调用其唯一实例的onNewIntent()方法。  
> **总结**：standard和singleTop都是一个任务栈可以有多个实例，每个实例可以属于不同的任务栈，但singleTop是当前任务栈栈顶存在它时不会创建新实例；而singleTask和singleInstance均是仅一个实例且仅属于一个任务栈，但singleInstance的任务栈中只有它一个activity。尽管singleTask可能属于新的任务栈，singleInstance一定属于新的任务栈，返回按钮仍然能返回到前一个activity。
  
**由于singleTask的性质，会存在这样的情况。**
![](https://developer.android.google.cn/images/fundamentals/diagram_backstack_singletask_multiactivity.png)

### 使用Intent flags
> 当启动一个Activity时，可以设定startActivity()的参数Intent的flags。
1. FLAG_ACTIVITY_NEW_TASK：效果与launchMode的singleTask一致
2. FLAG_ACTIVITY_SINGLE_TOP：效果与launchMode的singelTop一致
3. FLAG_ACTIVITY_CLEAR_TOP：

## 处理 affinities
> **affinities**表示一个Activity倾向于属于哪个任务栈。默认情况下，来自同一应用程序的所有活动都具有同一affinities。 所以，默认情况下，同一个应用程序中的所有活动都倾向于在同一个任务中。但是，您可以修改某个活动的默认affinity。 在不同应用中定义的活动可以共享一个affinity，或者在同一个应用中定义的活动可以分配不同的任务affinities。  
> 可以通过设定清单文件中的<activity>元素的taskAffinity 属性来修改任一activity的affinity。
> taskAffinity属性接受一个字符串值，其值为包名形式。

### Affinity在两种情况下发挥作用
1. 启动activity的的intent包含FLAG_ACTIVITY_NEW_TASK值的flag，或者该activity的launchMode属性为singleTask
此时该activity会运行于名字和taskAffinity属性相同的任务栈中。
2. 
