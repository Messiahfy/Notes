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
&emsp;&emsp;引用待操作数据的URI（Uri对象）和/或该数据的MIME类型，用于操作。提供的数据类型通常由Intent的Action决定。例如，如果Action是ACTION_EDIT，则数据应包含待编辑文档的URI。  
&emsp;&emsp;创建Intent时，除了指定URI以外，指定数据类型（其MIME类型）往往也很重要。例如，能够显示图像的Activity可能无法播放音频文件，即便URI
格式十分类似时也是如此。因此，指定数据的MIME类型有助于Android系统找到接收Intent的最佳组件。但有时，MIME类型可以从URI中推断得出，特别是数据是content:URI时尤其如此。
这表明数据位于设备中，且由ContentProvider控制，这使得数据MIME类型对系统可见。要仅设置数据URI，调用setData()。要仅设置MIME类型，调用setType()。如果要同时设置，则应调用setDataAndType()，因为setData()和setType()会互相抵消彼此的值。
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
用户可以选取共享内容所使用的应用。在调用startActivity()应该用Intent的resolveActivity()来判断至少有一个应用可以处理该Intent。另外，PackageManager的query...()可以返回接受隐式Intent的所有组件，resolve...()可以返回接受隐式Intent的最佳组件。Intent的resolveActivity()内部就调用了PackageManager的resolveActivity()。
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
&emsp;&emsp;要公布应用可以接收哪些隐式Intent，请在清单文件中使用\<intent-filter\>元素为每个应用组件声明一个或多个Intent过滤器。每个Intent过滤器
均根据Intent的Action、Data和Category指定自身接受的Intent类型。仅当隐式Intent可以通过Intent过滤器之一传递时，系统才会将该Intent传递给应用组件。
> 显示Intent始终会传递给其目标，无论组件声明的Intent过滤器如何均是如此。
&emsp;&emsp;接收隐式Intent的应用组件应当为自身可执行的每个作业声明各自的过滤器。例如，图像库应用中的一个Activity可能会有两个过滤器，分别用于查看
图像和编辑图像。当Activity启动时，它将检查Intent并根据Intent中的信息觉得具体的行为（例如，是否显示编辑器控件）。
&emsp;&emsp;每个Intent过滤器均有清单文件中的\<intent-filter\>元素定义，并嵌套在相应的应用组件（如Activity）中。在\<intent-filter\>内部，可以使用
以下三个元素中的一个或多个指定要接受的Intent类型。
* \<action\> 在name属性中声明接受的Intent操作。该值必须是操作的文本字符串（例如"android.intent.action.SEND"），而不是类常量（例如Intent.ACTION_SEND）
* \<data\> 使用一个或多个指定数据URI各个方面(scheme、host、port、path等)和MIME类型的属性，声明接收的数据类型。
* \<category\> 在name属性中，声明接受的Intent类别。该值必须是操作的文本字符串，而不是类常量。
> **注意**：为了接收隐式Intent，**必须**将 CATEGORY_DEFAULT 类别包括在Intent过滤器中。方法 startActivity() 和 startActivityForResult()
将按照Intent已经声明 CATEGORY_DEFAULT 类别的方式来处理所有Intent。如果未在Intent过滤器中声明此类别，则隐形Intent不会解析为您的Activity。
> 对于所有Activity，必须在清单文件中声明Intent过滤器。但是广播接收器的过滤器可以在清单文件中声明，也可以通过调用代码注册。
* ACTION_MAIN 操作指示这是主入口点，且不要求输入任何Intent数据。
* CATEGORY_LAUNCHER 类别指示此Activity的图标应放入系统的应用启动器。如果 \<activity\> 元素未使用icon指定图标，则系统使用 \<application\> 
元素中的图标。

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
&emsp;&emsp;\<intent-filter\>元素中必须至少包含一个\<action\>元素，如果\<intent-filter\>元素中没有\<action\>，那么过滤器不会接收任何隐式Intent。Intent的action必须存在且能够和过滤规则中的任何一个action相同即可匹配成功。
******************
* **Category** \<intent-filter\>元素中必须至少包含一个\<category android:name="android.intent.category.DEFAULT"/\>，否则无法接收任何隐式Intent。Intent如果设置了category，那么不管有几个，对于每个category来说，都必须是过滤器中声明了的某个category，过滤器声明的category数量可以多于Intent中指定的数量。由于系统调用startActivity()或者startActivityForResult()时会默认为Intent加上“android.intent.category.DEFAULT”这个category，且若要接收隐式Intent就必须在过滤器中加上"android.intent.category.DEFAULT"，所以Intent也可以不定义category，仍然可以匹配成功。
******************
* **Data**(URI和data type) Intent过滤器既可以不声明任何\<data\>元素，也可以声明多个此类元素。Data由两部分组成，mimeType和URI。\<intent-filter\>中的\<data\>中可以只声明mimeType，也可以只声明URI，或者两者同时声明。  
&emsp;&emsp;**mimeType** 指媒体类型，比如 image/jepg、audio/mpeg4-genric和video/*等，可以表示图片、文本、视频等不同的媒体格式。*表示任意MIME类型。
&emsp;&emsp;**URI** 包含四个部分：scheme, host, port 和 path，host可以在第一个字符使用*通配符，pathPattern也可以使用*通配符。结构如下：
```
<scheme>://<host>:<port>[<path>|<pathPrefix>|<pathPattern>]
```
&emsp;&emsp;虽然每个属性是可选的，但它们存在相关性：
* 如果没有指定scheme，则忽略所有其他URI属性，即整个URI无效。
* 如果指定了scheme，但没有指定host，则忽略port和所有path属性。  

&emsp;&emsp;将Intent中的URI与过滤器中的URI声明进行比较时，只将Intent的URI与过滤器中包含的URI的部分进行比较。例如：
* 如果一个过滤器只声明了scheme，那么只要Intent的URI的scheme部分与之相同就能与过滤器匹配。
* 如果过滤器指定了scheme、host和port但没有path，则Intent的URI的scheme、host和port部分与之相同都通过过滤器，而不管Intent的URI的path如何。
* 如果过滤器指定scheme、host、port和path，则只有具有相同scheme、host、port和path的URI才能通过过滤器。  

&emsp;&emsp;对Intent和过滤器中的URI和mimeType一起比较时，规则如下：
1. 过滤器没有指定任何URI或MIME类型：不包含URI和MIME类型的Intent才会通过测试。
2. 过滤器指定URI但没有指定MIME类型：包含URI但不包含MIME类型（既没有显式定义也不可从URI推导）的Intent在其URI与过滤器的URI格式匹配时通过测试
3. 过滤器指定MIME类型但没有指定URI：包含相同MIME类型但不包含URI的Intent，或者包含相同MIME类型且URI的scheme为content:或file:则会通过测试。 换句话说，如果组件的过滤器仅列出MIME类型，则假定组件支持content：和file：。
4. 过滤器同时指定URI和MIME类型：包含相同的MIME类型和URI的Intent才能通过测试。
 
&emsp;&emsp;包含在同一个\<intent-filter\>元素中的所有\<data\>元素将组合在一起。例如以下代码，Intent中的URI和MIME类型只要存在于其中即可。
intent.setDataAndType(Uri.parse("http://www.baidu.com"),"image/*");也能通过测试。
```
<activity android:name=".Main3Activity">
     <intent-filter>
         ...
         <data android:scheme="http" android:mimeType="image/*"/>
         <data android:scheme="tel" android:host="www.baidu.com" android:mimeType="audio/*"/>
     </intent-filter>
</activity>
```

> 可见：1.一个Activity只要能匹配任何一组intent-filter，即可成功启动对应的Activity； 2.要匹配任何一组intent-filter,就要同时匹配action，category和data才算是完全匹配； 另外在使用隐式调用要求IntentFilter必须定义action和category，data可以没有；其中 
category android:name=”android.intent.category.DEFAULT”是一定要设置的。因为启动的时候Intent会默认加上这个category，否则的话无法启动。
