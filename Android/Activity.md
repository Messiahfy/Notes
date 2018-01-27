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
> &emsp;&emsp;如果活动A启动活动B，则活动B可以在其清单文件中定义如何与当前任务相关联（如果有的话），活动A也可以通过Intent的flag请求活动B如何与当前任务相关联。 如果两个活动都定义活动B如何与任务相关联，则活动A的请求（如意图中定义的）覆盖活动B的请求（如其清单中所定义的）。  
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

&emsp;&emsp;Android浏览器应用应用声明网络浏览器Activity应该始终在浏览器应用自己的任务栈打开。  
  
&emsp;&emsp;由于singleTask的性质，会存在这样的情况。目前有两个任务栈，前台任务栈为1和2，后台任务栈为X和Y，且Y的启动模式为singleTask。若从前台任务栈启动Y，那么Y所在的整个任务栈都会被带到前台。（从singleTask的单实例且仅属于一个任务栈的性质，可以看出，若不将同任务栈的X一起带到前台，则Y或者X的任务栈将改变）  
&emsp;&emsp;下图如果是启动X，且X为singleTask，则X上的Y将出栈，只将X带到前台。
![singleTask特定情况图](https://developer.android.google.cn/images/fundamentals/diagram_backstack_singletask_multiactivity.png)

### 使用Intent flags
> 当启动一个Activity时，可以设定startActivity()的参数Intent的某些flags来修改默认的 activity与任务栈的联系。
1. FLAG_ACTIVITY_NEW_TASK：效果与launchMode的singleTask一致
2. FLAG_ACTIVITY_SINGLE_TOP：效果与launchMode的singelTop一致
3. FLAG_ACTIVITY_CLEAR_TOP：清除包含此Activity的Task中位于该Activity实例之上的其他Activity实例。这种行为的 launchMode 属性没有对应的值，只能通过代码设置。
如果被启动的Activity是standard：ABCD 启动 B ，会销毁B和B以上的实例，然后重新创建B，B 重新执行onCreate -> onStart，变成 AB。
配合FLAG_ACTIVITY_SINGLE_TOP使用，则 B 不会销毁只销毁B以上实例，然后B 执行onNewIntent -> onStart
配合FLAG_ACTIVITY_NEW_TASK则是singleTask效果。
4. FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS：具有这个标记的Activity不会出现在历史Activity的列表中，即多任务界面。
5. 其他FLAG以后再做查阅。

### startActivityForResult()对启动模式和任务栈的影响
&emsp;&emsp;使用startActivityForResult()的必要条件是被启动的Activity和原Activity要在同一个Task中。因此，使用startActivityForResult时，会强制将新启动的Activity放在原来的任务栈中，不论manifest清单文件中的<activiy>的xml属性 和 启动时设定的Intent.Flag_XXX(FLAG_ACTIVITY_NEW_TASK除外)。  
&emsp;&emsp;若Intent设定了FLAG_ACTIVITY_NEW_TASK，那么被启动的Activity将不会在原来的任务栈中运行，且原Activity的onActivityResult()将立即收到取消结果。此时startActivityForResult()相当于startActivity()。

## 处理关联(handling affinities)
> **affinities**表示一个Activity倾向于属于哪个任务栈。默认情况下，来自同一应用程序的所有活动都具有同一affinities。 所以，默认情况下，同一个应用程序中的所有活动都倾向于在同一个任务中。但是，您可以修改某个活动的默认affinity。 在不同应用中定义的活动可以共享一个affinity，或者在同一个应用中定义的活动可以分配不同的任务affinities。  
> 可以通过设定清单文件中的<activity>元素的taskAffinity 属性来修改任一activity的affinity。
> taskAffinity属性接受一个字符串值，其值为包名形式。

### taskAffinity在两种情况下发挥作用
1. 启动activity的的intent包含FLAG_ACTIVITY_NEW_TASK值的flag，或者该activity的launchMode属性为singleTask  
此时该activity会运行于名字和taskAffinity属性相同的任务栈中。
2. taskAffinity结合**allowTaskReparenting**使用，Activity将其 allowTaskReparenting 属性设置为"true"  
在这种情况下，Activity可以从启动它的任务栈移动到与其具有相同taskAffinity的任务栈（如果该相同taskAffinity的任务栈出现在前台）。例如，有两个应用分别是A和B，A启动了B中的一个Activity C，C最初所属的任务栈是应用A中启动Activity C的那个任务栈。但是当应用B的任务栈出现在前台，那么C将移动到应用B的任务栈（比如在A启动C后，点击home键，再点击B的启动图标，此时不是启动B的主activity，而是重新显示已经被启动的C）。可以这么理解，A启动了C，所以C运行于A的任务栈中，但因为默认taskAffinity是各自包名，所以当B启动后创建自己的任务栈，此时系统发现C的任务栈已经创建，则将C从A的任务栈中转移过来。

## 清理返回栈
> 如果用户长时间离开任务，则系统会清理除根Activity以外的所有Activity的任务。当用户再次返回到任务时，仅恢复根Activity。系统这样做的原因是，经过很长一段时间后，用户可能已经放弃了之前执行的操作，返回到任务是要开始执行新的操作。  
&emsp;&emsp;可以使用下列几个Activity属性修改此行为：
1. alwaysRetainTaskState 如果在任务栈的根Activity中将此属性设为"true"，则不会发生刚才所述的默认行为。即使在很长一段时间后，任务仍将所有Activity保留在栈中。
2. clearTaskOnLaunch 如果在任务栈的根Activity中将此属性设为"true"，则每当用户离开任务然后返回时，系统都会将任务栈清除到只剩下根Activity。
3. finishOnTaskLaunch 此属性类似于clearTaskOnLaunch，但它对单个Activity起作用，而非整个任务。

## 启动任务
