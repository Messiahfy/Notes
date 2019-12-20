# Services
&emsp;&emsp;Service 是一个可以在后台执行长时间运行操作而不提供用户界面的应用组件。服务可由其他应用组件启动，而且即使用户切换到其他应用，服务仍将在后台
继续运行。 此外，组件可以绑定到服务，以与之进行交互，甚至是执行进程间通信 (IPC)。 例如，服务可以处理网络事务、播放音乐，执行文件 I/O 或与内容提供程序交
互，而所有这一切均可在后台进行。  

服务基本上分为两种形式：  

* **started(启动)** 当应用组件（如 Activity）通过调用 startService() 启动服务时，服务即处于“启动”状态。一旦启动，服务即可在后台无限期运行，即使启动服
务的组件已被销毁也不受影响。 已启动的服务通常是执行单一操作，而且不会将结果返回给调用方。例如，它可能通过网络下载或上传文件。 操作完成后，服务会自行停止
运行。  

* **Bound(绑定)** 当应用组件通过调用 bindService() 绑定到服务时，服务即处于“绑定”状态。绑定服务提供了一个客户端-服务器接口，允许组件与服务进行交互、发
送请求、获取结果，甚至是利用进程间通信 (IPC) 跨进程执行这些操作。 仅当与另一个应用组件绑定时，绑定服务才会运行。 多个组件可以同时绑定到该服务，但全部取消绑定
后，该服务才会被销毁。

> 服务可以前台运行或者后台运行。前台服务执行一些对用户来说很明显的操作。 例如，音频应用程序会使用前台服务播放音轨。 前台服务必须显示状态栏图标。 即使用户
未与应用程序交互，前台服务也会继续运行。后台服务执行用户不直接注意的操作。 例如，如果应用程序使用服务来压缩其存储，那通常是后台服务。  
> **注意**：如果您的应用程序的目标是API级别26或更高，则当应用程序本身不在前台时，系统会对[运行后台服务施加限制](https://developer.android.google.cn/about/versions/oreo/background.html)。 在大多数情况下，您的应用程序应该使用
[scheduled job](https://developer.android.google.cn/topic/performance/scheduling.html)(预定作业)。  

&emsp;&emsp;虽然本文档是分开概括讨论这两种服务，但是您的服务可以同时以这两种方式运行，也就是说，它既可以是启动服务（以无限期运行），也允许绑定。问题只是
在于您是否实现了一组回调方法：onStartCommand()（允许组件启动服务）和 onBind()（允许绑定服务）。  

&emsp;&emsp;无论应用是处于启动状态还是绑定状态，或同时处于启动和绑定状态，任何应用组件均可像使用 Activity 那样通过调用 Intent 来使用服务（即使此服务来
自另一应用）。不过，您可以通过清单文件将服务声明为私有服务，并阻止其他应用访问。 使用清单文件声明服务部分将对此做更详尽的阐述。  
> 注意：服务在其托管进程的主线程中运行，它既不创建自己的线程，也不在单独的进程中运行（除非另行指定）。 这意味着，如果服务将执行任何 CPU 密集型工作或阻止
性操作（例如 MP3 播放或联网），则应在服务内创建新线程来完成这项工作。通过使用单独的线程，可以降低发生“应用无响应”(ANR) 错误的风险，而应用的主线程仍可继
续专注于运行用户与 Activity 之间的交互。  

## 基础知识
&emsp;&emsp;要创建服务，您必须创建 Service 的子类（或使用它的一个现有子类）。在实现中，您需要重写一些回调方法，以处理服务生命周期的某些关键方面并提供一种机制将组件
绑定到服务（如适用）。 应重写的最重要的回调方法包括：  

**onStartCommand\(\)** 当另一个组件（如 Activity）通过调用 startService() 请求启动服务时，系统将调用此方法，即使该服务已被启动。一旦执行此方法，服务即会启动并可在后台
无限期运行。 如果您实现此方法，则在服务工作完成后，需要由您通过调用 stopSelf() 或 stopService() 来停止服务。（如果您只想提供绑定，则无需实现此方法。）  

**onBind\(\)** 当另一个组件想通过调用 bindService() 与服务绑定（例如执行 RPC）时，系统将调用此方法。在此方法的实现中，您必须通过返回 IBinder 提供一个
接口，供客户端用来与服务进行通信。请务必实现此方法，但如果您并不希望允许绑定，则应返回 null。  

**onCreate\(\)** 首次创建服务时，系统将调用此方法来执行一次性设置程序（在调用 onStartCommand() 或 onBind() 之前）。如果服务已在运行，则不会调用此方法。  

**onDestroy()** 当服务不再使用且将被销毁时，系统将调用此方法。服务应该实现此方法来清理所有资源，如线程、注册的侦听器、接收器等。 这是服务接收的最后一个调用。

&emsp;&emsp;如果组件通过调用 startService() 启动服务（这会导致对 onStartCommand() 的调用），则服务将一直运行，直到服务使用 stopSelf() 自行停止运行，或由其他组件通过调用 stopService() 停止它为止。  

&emsp;&emsp;如果组件是通过调用 bindService() 来创建服务（且未调用 onStartCommand()，则服务只会在该组件与其绑定时运行。一旦该服务与所有客户端之间的绑定全部取消，系统便会销毁它。

&emsp;&emsp;仅当内存过低且必须回收系统资源以供具有用户焦点的 Activity 使用时，Android 系统才会强制停止服务。如果将服务绑定到具有用户焦点的 Activity，则它不太可
能会终止；如果将服务声明为在前台运行（稍后讨论），则它几乎永远不会终止。或者，如果服务已启动并要长时间运行，则系统会随着时间的推移降低服务在后台任务列
表中的位置，而服务也将随之变得非常容易被终止；如果服务是启动服务，则您必须将其设计为能够妥善处理系统对它的重启。 如果系统终止服务，那么一旦资源变得再
次可用，系统便会重启服务（不过这还取决于从 onStartCommand() 返回的值，本文稍后会对此加以讨论）。如需了解有关系统会在何时销毁服务的详细信息，请参阅[进
程和线程](https://developer.android.google.cn/guide/components/processes-and-threads.html)文档。  

### 使用清单文件声明服务
如同 Activity（以及其他组件）一样，您必须在应用的清单文件中声明所有服务。

要声明服务，请添加 <service> 元素作为 <application> 元素的子元素。例如：
```
<manifest ... >
  ...
  <application ... >
      <service android:name=".ExampleService" />
      ...
  </application>
</manifest>
```
如需了解有关使用清单文件声明服务的详细信息，请参阅 <service> 元素参考文档。

您还可将其他属性包括在 <service> 元素中，以定义一些特性，如启动服务及其运行所在进程所需的权限。android:name 属性是唯一必需的属性，用于指定服务的类名
。应用一旦发布，即不应更改此类名，如若不然，可能会存在因依赖显式 Intent 启动或绑定服务而破坏代码的风险（请阅读博客文章Things That Cannot Change[不能
更改的内容]）。

> **注意**：为了确保应用的安全性，请始终使用显式 Intent 启动或绑定 Service，且不要为服务声明 Intent 过滤器。使用隐式Intent启动服务会带来安全隐患，因为您无法确定将响应该Intent的服务，并且用户无法看到启动哪项服务。 从Android 5.0（API级别21）开始，如果您以隐式意图调用bindService（），则系统会引发异常。

此外，还可以通过添加 android:exported 属性并将其设置为 "false"，确保服务仅适用于您的应用。这可以有效阻止其他应用启动您的服务，即便在使用显式 Intent 时也如此。
> **注意**：用户可以查看其设备上正在运行的服务。 如果他们看到他们不认识或不信任的服务，他们可以停止服务。 为了避免用户意外停止服务，您需要将android：description属性添加到应用清单中的<service>元素。 在描述中，提供一个简短的语句来解释服务的作用以及它提供的好处。

## 创建started(启动)服务
&emsp;&emsp;启动服务由另一个组件通过调用 startService() 启动，这会导致调用服务的 onStartCommand() 方法。

&emsp;&emsp;服务启动之后，其生命周期即独立于启动它的组件，并且可以在后台无限期地运行，即使启动服务的组件已被销毁也不受影响。 因此，服务应通过调用 stopSelf() 结束
工作来自行停止运行，或者由另一个组件通过调用 stopService() 来停止它。

&emsp;&emsp;应用组件（如 Activity）可以通过调用 startService() 方法并传递 Intent 对象（指定服务并包含待使用服务的所有数据）来启动服务。服务通过 onStartCommand() 方法接收此 Intent。

&emsp;&emsp;例如，假设某 Activity 需要将一些数据保存到在线数据库中。该 Activity 可以启动一个协同服务，并通过向 startService() 传递一个 Intent，为该服务提供要
保存的数据。服务通过 onStartCommand() 接收 Intent，连接到互联网并执行数据库事务。事务完成之后，服务会自行停止运行并随即被销毁。  

> **注意**：注意：默认情况下，服务与服务声明所在的应用运行于同一进程，而且运行于该应用的主线程中。 因此，如果服务在用户与来自同一应用的 Activity 进行
交互时执行密集型或阻止性操作，则会降低 Activity 性能。 为了避免影响应用性能，您应在服务内启动新线程。  

从传统上讲，您可以扩展两个类来创建启动服务：  
**Service** 这是适用于所有服务的基类。扩展此类时，必须创建一个用于执行所有服务工作的新线程，因为默认情况下，服务将使用应用的主线程，这会降低应用正在运行
的所有 Activity 的性能。  

**IntentService** 这是 Service 的子类，它使用工作线程逐一处理所有启动请求。如果您不要求服务同时处理多个请求，这是最好的选择。 您只需实现 onHandleIntent() 方法即可，该方法会接收每个启动请求的 Intent，使您能够执行后台工作。   

------------------------
onStartCommand（Intent intent, int flags, int startId）的三个参数含义如下：  
* intent：startService(Intent intent)中传递的intent。如果服务在其进程消失后重新启动，它可能为空
* flags：表示启动请求的方式，可选值有 0，START_FLAG_REDELIVERY，START_FLAG_RETRY，0代表没有，它们具体含义如下：
  * START_FLAG_REDELIVERY 表明intent是先前传递的intent的重新传递，因为服务在先前已经返回了 START_REDELIVER_INTENT ，
  * START_FLAG_RETRY 该flag代表当onStartCommand调用后一直没有返回值时，会尝试重新去调用onStartCommand()。
* startId：表示特定启动请求的唯一整数，与stopSelfResult(int startId)或 stopSelf(int startId) 配合使用，stopSelfResult 可以更安全地根据ID停止服务。`startId的用法具体可见【停止服务】`。Service存活期间（onDestroy之前），收到的每个启动请求都有一个不同的startId。

&emsp;&emsp;请注意，onStartCommand() 方法必须返回整型数。整型数是一个值，用于描述系统应该如何在服务被系统杀死的情况下继续运行服务（如上所述，IntentService 的默
认实现将为您处理这种情况，不过您可以对其进行修改）。从 onStartCommand() 返回的值必须是以下常量之一：  

**START_NOT_STICKY**   如果服务的进程在 onStartCommand() 返回后（服务处于started状态）被杀死，且没有其他Intent来启动它，则该服务会退出started状态，并且只有明确调用startService才会重新创建它。该服务不会收到Intent为null的 onStartCommand() 调用，因为不会在没有Intent启动它时被重新创建。这是最安全的选项，可以避免在不必要时以及应用能够轻松重启所有未完成的作业时运行服务。  

**START_STICKY**    如果服务的进程在 onStartCommand() 返回后（服务处于started状态）被杀死，则会保留服务的started状态但不会保留已发送的Intent。之后，系统会尝试重新创建服务。因为服务处于started状态，所以会保证在创建新的服务后调用其 onStartCommand() 方法。如果这个服务没有收到其他让它启动的命令，那么这个方法被系统调用的时候，intent参数就是null。这适用于不执行命令、但无限期运行并等待作业的媒体播放器（或类似服务）。  

**START_REDELIVER_INTENT**    如果服务的进程在 onStartCommand() 返回后（服务处于started状态）被杀死，则会计划重建服务，并将最后一个传递的 Intent （即最近一次启动Service的Intent）通过 onStartCommand()重新传递。在服务使用 onStartCommand() 传递的startId调用 stopSelf(int startId) 前（或者直接调用stopSelf()），此Intent将保持并用于可能的重新传递，即没有主动stop就被销毁则会重建并传递Intent。重建时 onStartCommand() 方法不会传递为null的Intent，因为只有在所有启动此服务的都执行完的情况下才不会重启，所以多个Intent可能会有多次 onStartCommand() 的回调。这适用于主动执行应该立即恢复的作业（例如下载文件）的服务。

## 启动服务
您可以通过将 Intent（指定要启动的服务）传递给 startService()，从 Activity 或其他应用组件启动服务。Android 系统调用服务的 onStartCommand() 方法，并向其传递 Intent。（切勿直接调用 onStartCommand()。）  

例如，Activity 可以结合使用显式 Intent 与 startService()，启动上文中的示例服务 (HelloService)：
```
Intent intent = new Intent(this, HelloService.class);
startService(intent);
```
startService() 方法将立即返回，且 Android 系统调用服务的 onStartCommand() 方法。如果服务尚未运行，则系统会先调用 onCreate()，然后再调用 onStartCommand()。  

如果服务亦未提供绑定，则使用 startService() 传递的 Intent 是应用组件与服务之间唯一的通信模式。但是，如果您希望服务返回结果，则启动服务的客户端可以为广播创建一个 PendingIntent （使用 getBroadcast()），并通过启动服务的 Intent 传递给服务。然后，服务就可以使用广播传递结果。  

多个服务启动请求会导致多次对服务的 onStartCommand() 进行相应的调用。但是，要停止服务，只需一个服务停止请求（使用 stopSelf() 或 stopService()）即可。

## 停止服务
启动(started)服务必须管理自己的生命周期。也就是说，除非系统必须回收内存资源，否则系统不会停止或销毁服务，而且服务在 onStartCommand() 返回后会继续运行。因此，服务必须通过调用 stopSelf() 自行停止运行，或者由另一个组件通过调用 stopService() 来停止它。  

一旦请求使用 stopSelf() 或 stopService() 停止服务，系统就会尽快销毁服务。  

但是，如果服务同时处理多个 onStartCommand() 请求，则您不应在处理完一个启动请求之后停止服务，因为您可能已经收到了新的启动请求（在第一个请求结束时停止服务会终止第二个请求）。为了避免这一问题，您可以使用 stopSelf(int) 确保服务停止请求始终基于最近的启动请求。也就说，在调用 stopSelf(int) 时，传递与停止请求的 ID 对应的启动请求的 ID（传递给 onStartCommand() 的 startId）。然后，如果在您能够调用 stopSelf(int) 之前服务收到了新的启动请求，ID 就不匹配，服务也就不会停止。  

Service存活期间（onDestory之前），每个启动请求， onStartCommand() 中的 startId 都是唯一的。比如有三个启动请求，分别的startId为1、2、3。那么如果调用stopSelf(1)或者stopSelf(2)，Service并不会销毁，而是在调用stopSelf(3)时，也就是传入最后启动的startId，才会销毁。而如果直接调用stopSelf()，则会直接销毁。

> 注意：为了避免浪费系统资源和消耗电池电量，应用必须在工作完成之后停止其服务。 如有必要，其他组件可以通过调用 stopService() 来停止服务。即使为服务启用了绑定，一旦服务收到对 onStartCommand() 的调用，您始终仍须亲自停止服务。  

## 创建绑定服务
绑定服务允许应用组件通过调用 bindService() 与其绑定，以便创建长期连接（通常不允许组件通过调用 startService() 来启动它）。  

如需与 Activity 和其他应用组件中的服务进行交互，或者需要通过进程间通信 (IPC) 向其他应用公开某些应用功能，则应创建绑定服务。  

要创建绑定服务，必须实现 onBind() 回调方法以返回 IBinder，用于定义与服务通信的接口。然后，其他应用组件可以调用 bindService() 来检索该接口，并开始对服务调用方法。服务只用于与其绑定的应用组件，因此如果没有组件绑定到服务，则系统会销毁服务（您不必按通过 onStartCommand() 启动的服务那样来停止绑定服务）。  

要创建绑定服务，首先必须定义指定客户端如何与服务通信的接口。 服务与客户端之间的这个接口必须是 IBinder 的实现，并且服务必须从 onBind() 回调方法返回它。一旦客户端收到 IBinder，即可开始通过该接口与服务进行交互。  

多个客户端可以同时绑定到服务。客户端完成与服务的交互后，会调用 unbindService() 取消绑定。一旦没有客户端绑定到该服务，系统就会销毁它。  

绑定服务参考[官方绑定服务指南](https://developer.android.google.cn/guide/components/bound-services.html)

bindService 方法

第二个参数： ServiceConnection 接口有四个方法

第三个参数：如下为bindService第三个参数的具体分析

可以使用|位运算符设置多个flag
* `BIND_ABOVE_CLIENT` 表示绑定到此服务的客户端应用程序认为该服务比应用程序本身更重要。设置后，当绑定服务期间遇到OOM需要杀死进程，客户进程会先于服务进程被杀死，尽管不能保证确实如此。
* `BIND_ADJUST_WITH_ACTIVITY` 如果与Activity绑定，则基于Activity是否对用户可见，允许提高目标服务的过程重要性，使Service的重要性和Activity一致。
* `BIND_ALLOW_OOM_MANAGEMENT` 允许承载绑定服务的进程进行正常的内存管理。允许系统在内存不足或其他情况下杀死它，并且在其长时间运行的情况下更可能使其成为被杀死（并重启）的候选进程。
* `BIND_AUTO_CREATE` 只要绑定存在，就自动创建服务。注意，虽然这将创建服务，但它的service.onStartCommand（Intent，int，int）方法仍然只会由于对startService（Intent）的显式调用而被调用。但是，即使没有这个功能，在创建服务时，它仍然提供对服务对象的访问。设置后，只有绑定的Activity位于前台，此服务才会变得重要（影响低内存杀进程）。要影响系统对其重要性的判断，可以使用BIND_ADJUST_WITH_ACTIVITY。
* `BIND_DEBUG_UNBIND` 包括对不匹配的解除绑定调用的调试帮助。设置此标志时，将保留以下unbindService（ServiceConnection）调用的调用堆栈，以便在以后进行不正确的unbind调用时打印。请注意，这样做需要保留在应用程序生命周期内生成的绑定的信息，从而导致泄漏——这只应用于调试。
* `BIND_EXTERNAL_SERVICE` 让Service可以绑定并运行在调用方的App中，而不是在声明这个Service的App中，此Service还同时须要设置isolatedProcess为true。
* `BIND_IMPORTANT` 此服务对客户端非常重要，因此在客户端处于前台进程级别时应将其带到前台进程级别。不使用此flag的话，进程只能由客户端提升到可见性级别，即使该客户端位于前台。
* `BIND_INCLUDE_CAPABILITIES` 如果发起绑定的应用程序由于其前台状态（如活动或前台服务）而具有特定功能，则此标志将允许被绑定的应用程序获得相同的功能，只要它也具有所需的权限。
*`BIND_NOT_FOREGROUND` 不允许此绑定将目标服务的进程提升到前台调度优先级。它仍将被提升到至少与客户端相同的内存优先级（这样在客户端不可杀死的任何情况下，其进程都不会被杀死），但出于CPU调度的目的，它可能会留在后台。这只会在绑定客户端是前台进程而目标服务是后台进程的情况下产生影响。
* `BIND_NOT_PERCEPTIBLE` 如果绑定来自可见或用户可感知的应用程序，会将目标服务的重要性降低到可感知级别以下。这允许系统（临时）从内存中删除绑定进程，为更重要的用户可感知进程腾出空间。
* `BIND_WAIVE_PRIORITY` 不要影响目标服务宿主进程的调度或内存管理优先级。允许在后台LRU列表上管理服务的进程，就像在后台管理常规应用程序进程一样。

## 向用户发送通知
一旦运行起来，服务即可使用 Toast 通知或 状态栏通知 来通知用户所发生的事件。  

## 在前台运行服务
前台服务被认为是用户主动意识到的一种服务，因此在内存不足时，系统也不会考虑将其终止。 前台服务必须为状态栏提供通知，放在“正在进行”标题下方，这意味着除非服务停止或从前台移除，否则不能清除通知。  

例如，应该将通过服务播放音乐的音乐播放器设置为在前台运行，这是因为用户明确意识到其操作。 状态栏中的通知可能表示正在播放的歌曲，并允许用户启动 Activity 来与音乐播放器进行交互。  

要请求让服务运行于前台，请调用 startForeground()。此方法采用两个参数：唯一标识通知的整型数和状态栏的 Notification。例如：
```
Notification notification = new Notification(R.drawable.icon, getText(R.string.ticker_text),
        System.currentTimeMillis());
Intent notificationIntent = new Intent(this, ExampleActivity.class);
PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, 0);
notification.setLatestEventInfo(this, getText(R.string.notification_title),
        getText(R.string.notification_message), pendingIntent);
startForeground(ONGOING_NOTIFICATION_ID, notification);
```
> 注意：提供给 startForeground() 的整型 ID 不得为 0。  
要从前台移除服务，请调用 stopForeground()。此方法需传入一个布尔值参数，指示是否也移除状态栏通知。 此方法不会停止服务。 但是，如果您在服务正在前台运行时将其停止，则通知也会被移除。  

## 管理服务生命周期
服务的生命周期比 Activity 的生命周期要简单得多。但是，密切关注如何创建和销毁服务反而更加重要，因为服务可以在用户没有意识到的情况下运行于后台。  

服务生命周期（从创建到销毁）可以遵循两条不同的路径：  
* 启动(started)服务  该服务在其他组件调用 startService() 时创建，然后无限期运行，且必须通过调用 stopSelf() 来自行停止运行。此外，其他组件也可以通过调用 stopService() 来停止服务。服务停止后，系统会将其销毁。
* 绑定服务  该服务在另一个组件（客户端）调用 bindService() 时创建。然后，客户端通过 IBinder 接口与服务进行通信。客户端可以通过调用 unbindService() 关闭连接。多个客户端可以绑定到相同服务，而且当所有绑定全部取消后，系统即会销毁该服务。 （服务不必自行停止运行。）

这两条路径并非完全独立。也就是说，您可以绑定到已经使用 startService() 启动的服务。例如，可以通过使用 Intent（标识要播放的音乐）调用 startService() 来启动后台音乐服务。随后，可能在用户需要稍加控制播放器或获取有关当前播放歌曲的信息时，Activity 可以通过调用 bindService() 绑定到服务。在这种情况下，除非所有客户端均取消绑定，否则 stopService() 或 stopSelf() 不会实际停止服务。

## 实现生命周期回调
与 Activity 类似，服务也拥有生命周期回调方法，您可以实现这些方法来监控服务状态的变化并适时执行工作。 以下框架服务展示了每种生命周期方法：
```
public class ExampleService extends Service {
    int mStartMode;       // indicates how to behave if the service is killed
    IBinder mBinder;      // interface for clients that bind
    boolean mAllowRebind; // indicates whether onRebind should be used

    @Override
    public void onCreate() {
        // The service is being created
    }
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // The service is starting, due to a call to startService()
        return mStartMode;
    }
    @Override
    public IBinder onBind(Intent intent) {
        // A client is binding to the service with bindService()
        return mBinder;
    }
    @Override
    public boolean onUnbind(Intent intent) {
        // All clients have unbound with unbindService()
        return mAllowRebind;
    }
    @Override
    public void onRebind(Intent intent) {
        // A client is binding to the service with bindService(),
        // after onUnbind() has already been called
    }
    @Override
    public void onDestroy() {
        // The service is no longer used and is being destroyed
    }
}
```
> 注：与 Activity 生命周期回调方法不同，您不需要调用这些回调方法的超类实现。
![](https://developer.android.google.cn/images/service_lifecycle.png)  

通过实现这些方法，您可以监控服务生命周期的两个嵌套循环：  
* 服务的整个生命周期从调用 onCreate() 开始起，到 onDestroy() 返回时结束。与 Activity 类似，服务也在 onCreate() 中完成初始设置，并在 onDestroy() 中释放所有剩余资源。例如，音乐播放服务可以在 onCreate() 中创建用于播放音乐的线程，然后在 onDestroy() 中停止该线程。  

> 无论服务是通过 startService() 还是 bindService() 创建，都会为所有服务调用 onCreate() 和 onDestroy() 方法。  

* 服务的有效生命周期从调用 onStartCommand() 或 onBind() 方法开始。每种方法均有 {Intent 对象，该对象分别传递到 startService() 或 bindService()。  
对于启动服务，有效生命周期与整个生命周期同时结束（即便是在 onStartCommand() 返回之后，服务仍然处于活动状态）。对于绑定服务，有效生命周期在 onUnbind() 返回时结束。  
> **注**：尽管启动服务是通过调用 stopSelf() 或 stopService() 来停止，但是该服务并无相应的回调（没有 onStop() 回调）。因此，除非服务绑定到客户端，否则在服务停止时，系统会将其销毁 — onDestroy() 是接收到的唯一回调。  

上图说明了服务的典型回调方法。尽管该图分开介绍通过 startService() 创建的服务和通过 bindService() 创建的服务，但是请记住，不管启动方式如何，任何服务均有可能允许客户端与其绑定。因此，最初使用 onStartCommand()（通过客户端调用 startService()）启动的服务仍可接收对 onBind() 的调用（当客户端调用 bindService() 时）。
