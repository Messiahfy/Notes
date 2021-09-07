## 布局
### 代码布局
Frame布局：UIView的Frame用于代码中构建布局，基于父布局的坐标；bounds为自身的宽高，以及bounds.origins可以做偏移
autoresizing
AutoLayout Constraint布局

### Storyboard
可视化布局。容易产生冲突，不一定使用

## View Controller
一个view controller是UIViewController的子类，view controller管理视图层次结构。 它负责创建构成层次结构的视图对象，并负责处理与层次结构中的视图对象相关的事件。作为UIViewController的子类，都继承了一个重要的属性：
```
var view: UIView!
```
此属性指向UIView实例，该实例是视图控制器的视图层次结构的根。将视图控制器的view添加为窗口的子视图时，将添加视图控制器的整个视图层次结构。  

view controller有两种创建视图层次结构的方式：
1. 在Interface Builder中，通过使用storyboard等界面文件
2. 以编程方式，通过重写UIViewController方法loadView()。重写viewDidLoad()则是在View层次结构已经创建后，再另外添加或者修改

一个storyBoard可以有多个view controller，但每个storyBoard只能有一个initial view controller，initial view controller作为storyBoard的入口。在interface builder中可以另外拖一个view controller到storyBoard中。

**UIWindow**有一个rootViewController属性。将视图控制器设置为窗口的rootViewController时，该视图控制器的view将添加到window的视图层次结构中。设置此属性后，将删除window上所有现有的子视图，并使用适当的“自动布局”约束将视图控制器的view添加到窗口中。

每个应用都有一个main interface（主界面），即对storyBoard的引用，当应用程序启动时，main interface的initial view controller将设置为窗口的rootViewController。main interface可以在项目设置的Deployment Info中设置。默认是Main，对应Main.storyboard

view controller可以嵌套，**UITabBarController**可以用于切换view controller。**UITabBarController**持有一个view controller的数组，它还在屏幕底部维护一个tab bar（标签栏），并为其数组中的每个视图控制器提供一个tab。可以选择一个view controller，然后在Editor菜单中选择Embed in的Tab Bar Controller，就能让当前view controller添加到**UITabBarController**的view controller数组中，然后**UITabBarController**底部的tab就可以用于切换view controller。也可以手动添加**UITabBarController**并自行调整引用关系，control-drag连接**UITabBarController**和另一个view controller。

**UITabBarController**本身是UIViewController的子类。 UITabBarController的视图是具有两个子视图的UIView：tab bar和所选视图控制器的view。  
标签栏上的每个tab都可以显示标题和图像，并且每个视图控制器为此都维护一个tabBarItem属性。 当UITabBarController包含视图控制器时，其tabBarItem将显示在选项卡栏中。tabBarItem是由它表示的那个view controller来持有，而tabBar是由**UITabBarController**持有。tabBarItem的属性可以在storyboard中设置，也可以代码中设置。


通常，您将需要对Interface Builder中定义的子视图进行一些额外的初始化或配置，然后才能向用户显示。 那么在哪里可以访问子视图？ 有两个主要选项，具体取决于您需要执行的操作。 第一个选项是`viewDidLoad()`方法。加载视图控制器的界面文件后，将调用此方法，届时所有视图控制器的插座都将引用适当的对象。第二个选项是另一个UIViewController方法`viewWillAppear(_ :)`。 在将视图控制器的视图添加到窗口之前，将调用此方法。如果在应用程序运行期间仅需要配置一次，则覆盖`viewDidLoad()`。 如果您需要在每次视图控制器的视图出现在屏幕上时都需要进行配置，则覆盖`viewWillAppear(_ :)`。

#### 与view controller及其view进行交互
view controller及其view的生命周期中调用的一些方法:
* `init(coder:)`是view controller实例从storyboard中创建时的初始化方法，从storyboard创建view controller实例时，`init(coder:)`方法会被调用一次
* `init(nibName:bundle:)` 是UIViewController的指定初始化器。在不使用storyboard的情况下创建视图控制器实例时，其`init(nibName:bundle:)`会被调用一次。 请注意，在某些应用中，您可能最终会创建同一视图控制器类的多个实例。创建每个视图控制器时，将调用此方法一次。
* `loadView()`被重写可以以编程方式创建视图控制器的视图。
* `viewDidLoad()`被重写可以配置从界面文件创建的view。这个方法创建视图控制器的视图后被调用。
* `viewWillAppear(_:)`被重写以配置通过加载界面文件创建的view。每次将视图控制器显示到屏幕时，都会调用此方法和`viewDidAppear(_ :)`。每当将视图控制器移出屏幕时，都会调用`viewWillDisappear(_ :)`和`viewDidDisappear(_ :)`。

## 数据存储、加载、状态恢复
1. archiving是在iOS中持久保存模型对象的最常用方法之一。 归档对象涉及记录其所有属性并将其保存到文件系统。取消存档会根据该数据重新创建对象。（对象序列化）
2. 应用沙箱：每个iOS应用程序都有自己的应用程序沙箱。 应用程序沙箱是文件系统上与其他文件系统隔绝的目录。您的应用程序必须保留在其沙箱中，并且其他任何应用程序都不能访问其沙箱。  
应用程序沙箱包含许多目录：
    * **Documents/**  在此目录中，您可以写入应用程序在运行时生成的数据，并且要在应用程序的两次运行之间保持不变。 设备与iTunes或iCloud同步时将对其进行备份。 如果设备出现问题，则可以从iTunes或iCloud恢复此目录中的文件。
    * **Library/Caches/**  在此目录中，您可以写入应用程序在运行时生成的数据，并且要在应用程序的两次运行之间保持不变。 但是，与Documents目录不同，该设备与iTunes或iCloud同步时不会备份。 不备份缓存数据的主要原因是数据可能非常大，并延长了同步设备所需的时间。 可以将存储在其他位置（例如Web服务器）的数据放置在此目录中。 如果用户需要还原设备，则可以再次从Web服务器下载此数据。 如果设备磁盘空间不足，则系统可能会删除该目录的内容。
    * **Library/Preferences/**
    * **tmp/**

AppDelegate中有很多应用的生命周期函数。过渡到background状态是保存所有未完成的更改的好地方，因为这是您的应用程序在进入挂起状态之前最后一次执行代码。 一旦处于挂起状态，就可能根据操作系统的需要终止应用程序。

iOS系统架构 jianshu.com/p/80a27d111605
Instrument https://www.jianshu.com/p/4b882f1bd1a9
lldb调试 https://www.jianshu.com/p/7fb43e0b956a
cocoaPods https://juejin.cn/post/6844903731008536590

## 内存管理
不像Java、Kotlin系列语言使用可达性算法，Swift和Objective-C均使用引用计数法，所以需要另外手动处理循环引用的问题（使用弱引用等方式）。

引用计数的方式分为手动引用计数和自动引用计数，现在都是使用自动引用计数，需要额外了解的话，可以阅读一些iOS内存管理的历史变迁：
* [iOS内存管理](https://blog.devtang.com/2016/07/30/ios-memory-management/)
* https://juejin.cn/post/6844904129689714695

## 构建工具链
- Ruby：一门脚本语言，iOS开发的构建工具链会用到它。
- Gem：Ruby的包管理器。类似Node的npm，Python的pip。
- Cocoapods：Swift 和 Objective-C 的依赖管理工具，使用Ruby语言开发，本质就是是一个Gem包。描述项目依赖和构建目标的文件Podfile也是使用Ruby语言编写。
- Bundler：Bundler也是一个Gem包，用于管理当前目录下的Gem环境，例如Cocoapods的版本，达到统一版本的目的。和Android中的Gradle Wrapper作用类似。