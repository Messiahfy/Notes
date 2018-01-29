# Intent 和 Intent 过滤器
&emsp;&emsp;Intent是一个消息传递对象，可以使用它从其它应用组件请求操作。基本用例主要包括以下三个：
* **启动Activity**：在 startActivity() 中传递Intent对象，或者 startActivityForResult()和 onActivityResult()中。
* **启动服务**：将Intent传递给startService()或者bindService();Android 5.0(API 21)后，可以使用JobScheduler启动服务。
* **传播广播**：将Intent传递给sendBroadcast()，sendOrderedBroadcast()

## Intent 类型
* **显式Intent**：按名称（完全限定类名）指定要启动的组件。
* **隐式Intent**：不会指定特定的组件，而是声明要执行的常规操作，允许匹配IntentFilter的目标组件来处理它。
> &emsp;&emsp;创建显式Intent启动Activity或服务时，系统将立即启动Intent对象中指定的应用组件。  
&emsp;&emsp;创建隐式Intent时，系统通过将Intent上的内容与在设备上其他应用的清单文件中声明的Intent过滤器进行比较，从而找到要启动的相应组件。  
&emsp;&emsp;如果Intent与Intent过滤器匹配，则系统将启动该组件，并向其传递Intent对象。如果匹配多个Intent过滤器，则系统会显示一个对话框，支持用户选
取要使用的应用。

&emsp;&emsp;Intent过滤器是清单文件中的一个表达式，它指定组件要接收的Intent类型。通过对组件声明Intent过滤器，可以使其他应用能够直接使用某一特定类型
的Intent启动组件。
如果没有为组件声明任何Intent过滤器，则组件只能使用显式Intent启动。
> **注意**：为确保安全性，启动Service时，最好使用显式Intent。从Android 5.0（API 21）开始，如果使用隐式Intent调用bindService(),系统会引发异常。

## 构建Intent
&emsp;&emsp;Intent对象携带了Android系统用来确定要启动哪个组件的信息（例如，准确的组件名称（**Component name**）或应当接收该Intent的组件类别(**Category**)）
，以及收件人组件为了正确执行操作而使用的信息（例如，要采取的操作(**Action**)以及要处理的数据(**Data**)）  
&emsp;&emsp;Intent包含主要信息如下：
* **Component name**(组件名称)  
&emsp;&emsp;要启动的组件名称。这是构建显式Intent的一项重要信息，这意味着Intent应用仅传递给由组件名称定义的应用组件。如果没有组件名称，则Intent是隐
式的，且系统将根据其他Intent信息决定哪个组件应当接收Intent。  
&emsp;&emsp;可以使用Intent构造函数、 setComponent()、setClass()或setClassName()来设置组件名称。
* **Action**
&emsp;&emsp;指定要执行的通用操作（例如，view和pick）的字符串，只能指定一个。  
&emsp;&emsp;而对于广播Intent，这是指已经完成且正在报告的Action。  
&emsp;&emsp;操作在很大程度上决定了其余Intent的构成，特别是Data和extra中包含的内容。  
&emsp;&emsp;您可以指定自己的Action，供本应用或者其他应用调用组件。但是，通常使用由Intent类或其他框架类定义的Action常量。例如，Intent类的ACTION_VIEW、
ACTION_SEND；例如，对于在系统的设置应用中打开特定屏幕的操作，将在Setting中定义。
* **Data**
&emsp;&emsp;引用待操作数据和/或该数据MIME类型的URI（Uri对象）。提供的数据类型通常由Intent的Action决定。例如，如果Action是ACTION_EDIT，则数据
应包含待编辑文档的URI。  
&emsp;&emsp;创建Intent时，除了指定URI以外，指定数据类型（其MIME类型）往往也很重要。例如，能够显示图像的Activity可能无法播放音频文件，即便URI
格式十分类似时也是如此。因此，指定数据的MIME类型有助于Android系统找到接收Intent的最佳组件。但有时，MIME类型可以从URI中推断得出，特别是数据是content:URI时尤其如此。
这表明数据位于设备中，且由ContentProvider控制，这使得数据MIME类型对系统可见。要仅设置数据URI，调用setData()。要仅设置MIME类型，调用setType()。
如果要同时设置，则应调用setDataAndType()，因为setData()和setType()会互相抵消彼此的值。
* **Category**
&emsp;&emsp;一个包含应处理Intent的组件的类别的附加信息的字符串。可以将任意数量的category放入一个Intent中，但大多数Intent均不需要类别。可以使用
addCategory()来指定类别。可使用系统的标准类别或自定义的Category。  
************
*以上列出的属性（Component name、Action、Data、Category）表示Intent的既定特征。通过读取这些信息，Android系统能够解析应当启动哪个应用组件。但是
Intent也有可能携带不影响解析应用组件的信息，如下：*
* **Extras**
&emsp;&emsp;携带完成请求操作所需的附加信息的键值对。使用putExtra()添加键值对。或者创建一个包含所有extra数据的Bundle对象，然后使用Intent的putExtras()
方法将Bundle传入。可使用系统的标准Extra或自定义的Extra。不要使用 Parcelable 或 Serializable数据在Extra键值对中。
* **Flags**
&emsp;&emsp;调用setFlags()、addFlags()来设定系统启动Activity的启动模式及任务栈相关表现。

## 隐式Intent示例
&emsp;&emsp;隐式Intent指定能够在可以执行相应操作的设备上调用任何应用的操作。如果您的应用无法执行该操作而其他应用可以，且您希望用户选取要使用的应用，则使用隐式Intent
非常有用。  
&emsp;&emsp;例如，如果您希望用户与他人共享您的内容，请使用ACTION_SEND操作创建Intent，并添加指定共享内容的extra。使用该应用调用startActivity() 时，
用户可以选取共享内容所使用的应用。在调用startActivity()应该用Intent的resolveActivity()来判断至少有一个应用可以处理该Intent。
```
// Create the text message with a string
Intent sendIntent = new Intent();
sendIntent.setAction(Intent.ACTION_SEND);
sendIntent.putExtra(Intent.EXTRA_TEXT, textMessage);
sendIntent.setType("text/plain");

// Verify that the intent will resolve to an activity
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(sendIntent);
}
```
## 强制使用应用选择器
&emsp;&emsp;如果有多个应用响应隐式Intent，用户可以选择要使用的应用，并将其设置为该操作的默认选项。但是可以强制使用应用选择器，要求用户每次都要选择。  
&emsp;&emsp;要显示选择器，请使用createChooser()创建Intent，并将其传递给startActivity()。
```
Intent sendIntent = new Intent(Intent.ACTION_SEND);
...

// Always use string resources for UI text.
// This says something like "Share this photo with"
String title = getResources().getString(R.string.chooser_title);
// Create intent to show the chooser dialog
Intent chooser = Intent.createChooser(sendIntent, title);

// Verify the original intent will resolve to at least one activity
if (sendIntent.resolveActivity(getPackageManager()) != null) {
    startActivity(chooser);
}
```

## 接收隐式Intent
&emsp;&emsp;要公布应用可以接收哪些隐式Intent，请在清单文件中使用<intent-filter>元素为每个应用组件声明一个或多个Intent过滤器。每个Intent过滤器
均根据Intent的Action、Data和Category指定自身接受的Intent类型。仅当隐式Intent可以通过Intent过滤器之一传递时，系统才会将该Intent传递给应用组件。
> 显示Intent始终会传递给其目标，无论组件声明的Intent过滤器如何均是如此。
&emsp;&emsp;接收隐式Intent的应用组件应当为自身可执行的每个作业声明各自的过滤器。例如，图像库应用中的一个Activity可能会有两个过滤器，分别用于查看
图像和编辑图像。当Activity启动时，它将检查Intent并根据Intent中的信息觉得具体的行为（例如，是否显示编辑器控件）。
&emsp;&emsp;每个Intent过滤器均有清单文件中的<intent-filter>元素定义，并嵌套在相应的应用组件（如Activity）中。在<intent-filter>内部，可以使用
以下三个元素中的一个或多个指定要接受的Intent类型。
* \<action\> 在name属性中声明接受的Intent操作。该值必须是操作的文本字符串（例如"android.intent.action.SEND"），而不是类常量（例如Intent.ACTION_SEND）
* \<data\> 使用一个或多个指定数据URI各个方面(scheme、host、port、path等)和MIME类型的属性，声明接收的数据类型。
* \<category\> 在name属性中，声明接受的Intent类别。该值必须是操作的文本字符串，而不是类常量。
> **注意**：为了接收隐式Intent，**必须**将 CATEGORY_DEFAULT 类别包括在Intent过滤器中。方法 startActivity() 和 startActivityForResult()
将按照Intent已经声明 CATEGORY_DEFAULT 类别的方式来处理所有Intent。如果未在Intent过滤器中声明此类别，则隐形Intent不会解析为您的Activity。
> 对于所有Activity，必须在清单文件中声明Intent过滤器。但是广播接收器的过滤器可以在清单文件中声明，也可以通过调用代码注册。

## 使用pending intent
&emsp;&emsp;PendingIntent对象是Intent对象的包装器。官网：PendingIntent的主要目的是授权外部应用使用包含的Intent，就像是它从应用本身的进程中执行的一样。
个人理解：PendingIntent相当于异步执行的Intent。
&emsp;&emsp;PendingIntent的主要用例包括：
* 声明用户使用您的通知执行操作时所要执行的Intent（Android系统的NotificationManager执行Intent）
* 声明用户使用您的应用小部件(App Widget)执行操作时要执行的Intent（主屏幕应用（Home screen app）执行Intent）
* 声明未来某一特定时间要执行的Intent（Android系统的AlarmManager执行Intent）。  
由于每个Intent对象均设计为由特定类型的应用组件（Activity、Service或BroadcastReceiver）进行处理，所以也必须基于相同的考虑因素创建PendingIntent。
使用PendingIntent时，您的应用不会以调用例如startActivity()的这种方式来执行。而是通过调用相应的构建方法创建PendingIntent时，必须声明所需的组件类型：
* PendingIntent.getActivity() 适用于启动Activity的Intent
* PendingIntent.getService() 适用于启动Service的Intent
* PendingIntent.getBroadcast() 适用于启动BroadcastReceiver的Intent

## 隐式Intent的解析（匹配规则）
&emsp;&emsp;当系统收到隐式Intent以启动Activity时，它根据以下三个方面将该Intent与Intent过滤器进行比较，搜索该Intent的最佳Activity：
* **Action**
&emsp;&emsp;Intent过滤器既可以不声明任何<action>元素，也可以声明多个此类元素。如果过滤器声明了action，则Intent的action必须存在且能够和过滤规则中的任何一个action相同即可匹配成功。
* **Category** Intent过滤器既可以不声明任何<category>元素，也可以声明多个此类元素。Intent如果设置了category，那么不管有几个，对于每个category来说，都必须是过滤器中声明了的某个category，过滤器声明的category数量可以
多于Intent中指定的数量。由于系统调用startActivity()或者startActivityForResult()时会默认为Intent加上“android.intent.category.DEFAULT”这
个category，且若要接收隐式Intent就必须在过滤器中加上"android.intent.category.DEFAULT"，所以Intent也可以不定义category，仍然可以匹配成功。
* **Data**(URI和data type) Intent过滤器既可以不声明任何<data>元素，也可以声明多个此类元素。

