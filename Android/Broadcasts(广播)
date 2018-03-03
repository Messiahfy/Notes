# Broadcasts
Android应用可以发送或接收来自Android系统和其他Android应用的广播消息，类似于发布 - 订阅设计模式。 这些广播是在感兴趣的事件发生时发送的。 例如，Android
系统在发生各种系统事件时发送广播，例如系统启动或设备开始充电时。 应用程序还可以发送自定义广播，例如，向其他应用程序通知他们可能感兴趣的内容（例如，某些新
数据已被下载）。  

应用可以通过注册来接收特定的广播。 发送广播时，系统会自动将广播路由到已订阅接收该特定广播类型的应用。  

一般来说，广播可以用作应用程序之间和正常用户流程之外的消息传递系统。 但是，您必须小心，不要滥用机会在后台响应广播和运行作业，这可能会导致系统性能降低。

## （系统广播）System broadcasts
系统在发生各种系统事件时自动发送广播，例如系统切入和切出飞行模式。 系统广播被发送到订阅接收事件的所有应用程序。  

广播消息本身包装在一个Intent对象中，该对象的action字符串标识发生的事件（例如android.intent.action.AIRPLANE_MODE）。 inetnt还可能包括捆绑到其额外领域
的附加信息。 例如，飞机模式inetnt包含一个布尔extra，表示飞行模式是否打开。  

有关系统广播操作的完整列表，请参阅Android SDK中的BROADCAST_ACTIONS.TXT文件。 每个广播action都有一个与其相关的常量字段。 例如，常量
ACTION_AIRPLANE_MODE_CHANGED的值是android.intent.action.AIRPLANE_MODE。 每个广播action的文档可在其关联的常量字段中找到。

### 系统广播的变化
Android 7.0及更高版本不再发送以下系统广播。 此优化会影响所有应用，而不仅仅是针对Android 7.0的应用。
* ACTION_NEW_PICTURE
* ACTION_NEW_VIDEO  
面向Android 7.0（API级别24）及更高级别的应用必须使用registerReceiver（BroadcastReceiver，IntentFilter）注册以下广播。 在清单中声明接收者不起作用。
* CONNECTIVITY_ACTION  

从Android 8.0（API级别26）开始，系统对清单声明的接收方施加额外的限制。 如果您的应用定位到API级别26或更高级别，则无法使用清单为大多数隐式广播（不是专门针对您的应用的广播）声明接收方。

## （接收广播）Receiving broadcasts  
应用程序可以通过两种方式接收广播：通过清单声明的接收器和上下文注册（动态代码注册）的接收器。

### 清单声明广播接收器（Manifest-declared receivers）
如果您在清单中声明了广播接收器，系统会在广播发送时启动您的应用程序（如果应用程序尚未运行）。  
> **注意**：如果您的应用程序的目标API级别为26或更高，则不能使用清单来声明隐式广播的接收器（不是专门针对您的应用程序的广播），但可以[免除该限制的一些隐式广播](https://developer.android.google.cn/guide/components/broadcast-exceptions.html)除
外。 在大多数情况下，您可以使用 scheduled jobs 。  

要在清单中声明广播接收器，请执行以下步骤：  

1. 在应用程序的清单中指定<receiver>元素。  
```
<receiver android:name=".MyBroadcastReceiver"  android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
        <action android:name="android.intent.action.INPUT_METHOD_CHANGED" />
    </intent-filter>
</receiver>
```
intent filter指定您的接收器订阅的广播操作。  

2. 创建BroadcastReceiver的子类并实现onReceive（Context,Intent）。 以下示例中的广播接收机会记录并显示广播的内容：  
```
public class MyBroadcastReceiver extends BroadcastReceiver {
    private static final String TAG = "MyBroadcastReceiver";
    @Override
    public void onReceive(Context context, Intent intent) {
        StringBuilder sb = new StringBuilder();
        sb.append("Action: " + intent.getAction() + "\n");
        sb.append("URI: " + intent.toUri(Intent.URI_INTENT_SCHEME).toString() + "\n");
        String log = sb.toString();
        Log.d(TAG, log);
        Toast.makeText(context, log, Toast.LENGTH_LONG).show();
    }
}
```
系统包管理器(package manager)在安装应用程序时注册广播接收器。 然后接收器成为您的应用程序的单独入口点，这意味着系统可以启动应用程序并在应用程序
当前未运行时传送广播。  

系统创建一个新的BroadcastReceiver组件对象来处理它收到的每个广播。 此对象仅在调用onReceive（Context，Intent）期间有效。 一旦您的代码从此方法返回，
系统会认为该组件不再处于活动状态

### Context-registered receivers（上下文注册广播接收器，动态代码注册）
要使用上下文注册接收器，请执行以下步骤：  

1. 创建BroadcastReceiver的一个子类MyBroadcastReceiver实例（同样要创建该子类并实现onReceive（Context,Intent））。
```
BroadcastReceiver br = new MyBroadcastReceiver();
```
2. 创建一个IntentFilter并通过调用registerReceiver（BroadcastReceiver，IntentFilter）来注册接收器：
```
IntentFilter filter = new IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION);
filter.addAction(Intent.ACTION_AIRPLANE_MODE_CHANGED);
this.registerReceiver(br, filter);
```
> **注意**：要注册本地广播，请改为调用LocalBroadcastManager.registerReceiver（BroadcastReceiver，IntentFilter）。  
只要注册上下文有效，上下文注册的接收器就会接收广播。 例如，如果您在活动上下文中注册，只要活动未被销毁，您就会收到广播。 如果您注册了应用程序上下文，只
要应用程序正在运行，就会收到广播。
3. 要停止接收广播，请调用unregisterReceiver（android.content.BroadcastReceiver）。 当您不再需要接收器或上下文不再有效时，请务必注销接收器。  
注意你注册和注销接收者的位置，例如，如果你使用活动的上下文在onCreate（Bundle）中注册一个接收者，你应该在onDestroy（）中取消注册，以防止接收者泄漏出
活动上下文。 如果你在onResume（）中注册一个接收器，你应该在onPause（）中取消注册，以防止多次注册（如果你不想在暂停时接收广播，这可以减少不必要的系统
开销）。 不要在onSaveInstanceState（Bundle）中取消注册，因为如果用户移回到历史堆栈中，则不会调用它。

### 对进程状态的影响
您的BroadcastReceiver的状态（无论是否正在运行）会影响其包含进程的状态，从而可能影响其被系统杀死的可能性。 例如，当一个进程执行一个接收器（也就是说，
当前正在onReceive（）方法中运行该代码）时，它被认为是一个前台进程。 除非存在极大的内存压力，否则系统会继续运行。  

但是，一旦您的代码从onReceive（）返回，BroadcastReceiver不再处于活动状态。 广播接收器的主机进程变得与其中运行的其他应用程序组件一样重要。 如果该进程仅
托管清单声明的接收方（用户从未或最近未与之交互过的应用的常见情况），则从onReceive（）返回时，系统将其进程视为低优先级进程，并可能 杀死它以使资源可用
于其他更重要的进程。  

出于这个原因，您不应该从广播接收器开始长时间运行后台线程。 在onReceive（）之后，系统可以随时终止进程以回收内存，并且这样做会终止在进程中运行的衍生线
程。 为了避免这种情况，您应该调用goAsync（）（如果您需要更多时间在后台线程中处理广播）或使用JobScheduler从接收方调度JobService，那么系统知道该进程
继续执行活动 工作。 有关更多信息，请参阅进程和应用程序生命周期。  

以下代码片段显示了一个BroadcastReceiver，它使用goAsync（）来标记onReceive（）完成后需要更多时间才能完成。 如果你想在你的onReceive（）中完成的工作
足够长，导致UI线程错过一帧（> 16ms），这使得它更适合后台线程，这是特别有用的。
```
public class MyBroadcastReceiver extends BroadcastReceiver {
    private static final String TAG = "MyBroadcastReceiver";

    @Override
    public void onReceive(final Context context, final Intent intent) {
        final PendingResult pendingResult = goAsync();
        AsyncTask<String, Integer, String> asyncTask = new AsyncTask<String, Integer, String>() {
            @Override
            protected String doInBackground(String... params) {
                StringBuilder sb = new StringBuilder();
                sb.append("Action: " + intent.getAction() + "\n");
                sb.append("URI: " + intent.toUri(Intent.URI_INTENT_SCHEME).toString() + "\n");
                Log.d(TAG, log);
                // Must call finish() so the BroadcastReceiver can be recycled.
                pendingResult.finish();
                return data;
            }
        };
        asyncTask.execute();
    }
}
```

## 发送广播
Android为应用程序发送广播提供了三种方式：
* sendBroadcast（Intent）方法以未定义的顺序向所有接收器发送广播。这被称为普通广播。这样更有效率，但意味着接收器无法读取其他接收器的结果、传播从广播接收的数据或中止广播。
* sendOrderedBroadcast（Intent，String）方法每次向一个接收器发送广播。当每个接收器依次执行时，它可以将结果传播到下一个接收器，或者它可以完全中止广播，使其不会传递给其他接收器。运行的订单接收器可以通过匹配意图过滤器的android：priority属性进行控制;具有相同优先级的接收器将以任意顺序运行。
* LocalBroadcastManager.sendBroadcast方法将广播发送到与发件人位于同一应用程序中的接收器。如果您不需要跨应用程序发送广播，请使用本地广播。实现效率更高（不需要进程间通信），您不必担心与其他应用程序能够接收或发送广播有关的任何安全问题。

以下代码片段演示了如何通过创建一个Intent并调用sendBroadcast（Intent）来发送广播。
```
Intent intent = new Intent();
intent.setAction("com.example.broadcast.MY_NOTIFICATION");
intent.putExtra("data","Notice me senpai!");
sendBroadcast(intent);
```
广播消息包装在一个Intent对象中。 intent的action字符串必须提供应用程序的Java包名称语法并唯一标识广播事件。 您可以使用putExtra（String，Bundle）将附
加信息附加到intent。 您也可以通过在意图上调用setPackage（String）将广播限制在同一组织中的一组应用程序。
> **注意**：尽管intent既用于发送广播又用startActivity（Intent）开始活动，这些操作完全不相关。 广播接收器无法看到或捕获用于启动活动的意图; 同样，当
你广播一个意图时，你不能找到或开始一个活动。

## 限制带有权限的广播
权限允许您将广播限制为拥有特定权限的应用程序集。 您可以对广播的发送者或接收者实施限制。

### 发送广播时附带权限
当您调用sendBroadcast（Intent，String）或sendOrderedBroadcast（Intent，String，BroadcastReceiver，Handler，int，String，Bundle）时，您可以指
定权限参数。 只有在清单中通过标签请求许可的接收方才能接收广播，并且随后被授予许可（如果存在危险）。 例如，下面的代码发送一个广播：
```
sendBroadcast(new Intent("com.example.NOTIFY"),
              Manifest.permission.SEND_SMS);
```
要接收这个广播，接收方应用程序必须请求权限，如下所示：
```
<uses-permission android:name="android.permission.SEND_SMS"/>
```
您可以指定现有系统权限（如SEND_SMS），也可以使用<permission>元素定义自定义权限。 有关一般权限和安全性的信息，请参阅[系统权限](https://developer.android.google.cn/guide/topics/security/permissions.html)。
> **注意**：自定义权限是在安装应用程序时注册的。 定义自定义权限的应用程序必须安装在使用它的应用程序之前

### 接收广播时附加权限
如果在注册广播接收器时指定一个权限参数（或者使用registerReceiver（BroadcastReceiver，IntentFilter，String，Handler）或清单中的<receiver>标签），
那么只有使用<uses-permission> 标签（如果它是危险的，随后被授予许可）可以向接收方发送intent。

例如，假设您的接收应用程序具有清单声明的接收器，如下所示：
```
<receiver android:name=".MyBroadcastReceiver"
          android:permission="android.permission.SEND_SMS">
    <intent-filter>
        <action android:name="android.intent.action.AIRPLANE_MODE"/>
    </intent-filter>
</receiver>
```
或者您的接收应用程序有一个上下文注册的接收器，如下所示：
```
IntentFilter filter = new IntentFilter(Intent.ACTION_AIRPLANE_MODE_CHANGED);
registerReceiver(receiver, filter, Manifest.permission.SEND_SMS, null );
```
然后，为了能够向这些接收器发送广播，发送应用程序必须请求权限，如下所示：
```
<uses-permission android:name="android.permission.SEND_SMS"/>
```

## 安全考虑和最佳实践
以下是发送和接收广播的一些安全考虑事项和最佳做法：

* 如果您不需要将广播发送到应用以外的组件，则可以使用支持库中的LocalBroadcastManager发送和接收本地广播。 LocalBroadcastManager效率更高（不需要进程
间通信），并且可以避免考虑与其他应用程序能够接收或发送广播相关的任何安全问题。本地广播可以在您的应用程序中用作通用的发布/订阅事件总线，而无需系统广播
的任何开销。

* 如果许多应用程序已注册接收清单中的相同广播，则可能会导致系统启动大量应用程序，从而对设备性能和用户体验产生重大影响。为了避免这种情况，优先使用上下文
注册声明。有时，Android系统本身会强制使用上下文注册的接收器。例如，CONNECTIVITY_ACTION广播仅传递给上下文注册的接收器。

* 不要使用隐式Intent广播敏感信息。任何注册的应用程序都可以读取信息以接收广播。有三种方法可以控制谁可以接收您的广播：
 * 您可以在发送广播时指定权限。
 * 在Android 4.0及更高版本中，您可以在发送广播时使用setPackage（String）指定一个包。系统将广播限制为与包匹配的一组应用程序。
 * 您可以使用LocalBroadcastManager发送本地广播。
 
* 当您注册接收器时，任何应用程序都可能将潜在的恶意广播发送到您应用的接收器。有三种方法可以限制您的应用收到的广播：
 * 您可以在注册广播接收机时指定权限。
 * 对于清单声明的接收者，您可以在清单中将android：exported属性设置为“false”。接收器不接收来自应用程序外部的广播。
 * 您只能使用LocalBroadcastManager将自己限制为本地广播。
 
* 广播操作的命名空间是全局的。确保action名称和其他字符串被写入您拥有的名称空间中，否则您可能会无意中与其他应用程序发生冲突。

* 因为接收者的onReceive（Context，Intent）方法在主线程上运行，所以它应该快速执行并返回。如果您需要执行长时间运行的工作，请注意产生线程或启动后台服务
，因为系统会在onReceive（）返回后终止整个进程。有关更多信息，请参阅对进程状态的影响来执行长时间运行的工作，我们建议：
 * 在接收器的onReceive（）方法中调用goAsync（）并将BroadcastReceiver.PendingResult传递给后台线程。这使从onReceive（）返回后的广播保持活动状态。但是，
即使采用这种方法，系统也希望您能够很快完成广播（不到10秒）。它确实允许您将工作移至另一个线程以避免妨碍主线程。
 * 使用JobScheduler调度工作。有关更多信息，请参阅智能作业计划。
 
* 不要从广播接收机开始活动，因为用户体验很刺耳;特别是如果有多个接收器的话。相反，请考虑显示通知。




