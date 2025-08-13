## iOS应用启动流程
### main函数
我们首先搞清楚iOS应用的代码入口。UIApplicationMain函数的第3个参数`UIApplication`一般传`nil`，即使用默认的`UIApplication`类实例；第4个参数`UIApplicationDelegate`需要我们传入，一般都是默认工程中的`AppDelegate`。下面就是使用`Objective-C`创建的项目中的main函数：
```
int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    @autoreleasepool {
        // Setup code that might create autoreleased objects goes here.
        appDelegateClassName = NSStringFromClass([AppDelegate class]);
    }
    return UIApplicationMain(argc, argv, nil, appDelegateClassName);
}
```
通过`Objective-C`的项目，可以更清楚地了解这里执行的情况，而使用`swift`的情况，已经看不到`main`函数，也看不到调用`UIApplicationMain`：
```
@main
class AppDelegate: UIResponder, UIApplicationDelegate {
   //...省略内部的属性和函数等
}
```
这里使用了`@main`修饰了AppDelegate类（Xcode 12之前使用的是`@UIApplicationMain`），使用这个标签，就已经帮我们把手动调用`UIApplicationMain`的工作完成了。

如果使用swift，但是想手动来完成调用`UIApplicationMain`要怎么做呢？我们先删掉`@main`，并且创建`main.swift`文件（），代码如下：
```
UIApplicationMain(CommandLine.argc, CommandLine.unsafeArgv, nil, NSStringFromClass(AppDelegate))
```
> main.swift是swift的特殊文件，它将被作为程序的主入口，并且类似脚本语言从头到尾开始执行，而不用main函数

### 主要流程
1. 打开app执行到 main 函数
2. 执行`UIApplicationMain`函数，这里可以传入`UIApplication`和`UIApplicationDelegate`代理对象，函数内部执行如下工作：
    * 创建 UIApplication 实例
    * 设置 AppDelegate 委托
    * 根据 Info.plist 或代码配置，加载main storyboard、xib等（有的话）
    * 创建UIWindow和设置UIWindow的rootViewController（使用storyboard、xib的情况会自动完成）。iOS 13以上还会处理Scene相关配置
    * 函数内部会开启事件循环
    * 显示界面

### 初始视图的加载
前面了解了关于应用程序入口的情况后，现在来了解view加载的情况。
* Launch Screen
苹果现在要求iOS应用的启动页均使用storyboard，所以要修改启动页的界面，就在`LaunchScreen.storyborad`文件中去修改，这部分只能是静态的，不能使用代码编程。

* 主界面
启动页之后就需要显示主界面。

iOS 13前，AppDelegate需要设置window（UIWindow类型）属性，AppDelegate也需要管理UI生命周期；iOS 13开始，为了支持多窗口场景引入了SceneDelegate来管理应用程序的窗口和界面生命周期，如果使用Scene配置，window将放在SceneDelegate中，AppDelegate只管理应用的生命周期，UI的生命周期将在SceneDelegate中管理，AppDelegate的UI生命周期将不会被系统回调。**注**：这里说的UI生命周期是前后台切换之类比较宽泛的变化，UI具体的加载、可见这类的是在具体的ViewController中管理。

SceneDelegate与AppDelegate的关系
* iOS 13之前：
AppDelegate负责管理应用程序的生命周期和UIWindow。
* iOS 13及之后：
AppDelegate负责应用程序的生命周期（例如启动、终止）。
SceneDelegate负责场景的生命周期和UIWindow管理。

主界面分为使用`storyboard`和`代码`两种方式，如果`window`是`nil`，并且设置了`main storyboad`，那么会自动创建`window`，并设置`storyboard`对应的`ViewController`为window的`rootViewController`，则`ViewController`的`根view`也将放在`window`中。

如果是手动创建`window`，那么在`AppDelegate`的`willFinishLaunchingWithOptions`阶段，或者`SceneDelegate`的`willConnectTo`阶段，创建`UIWindow`，设置`rootViewController`，并调用`makeKeyAndVisible`。`AppDelegate`还是`SceneDelegate`取决于上文所说的iOS版本以及是否配置了Scene。
```
self.window = UIWindow(frame:UIScreen.main.bounds)
self.window!.backgroundColor = UIColor.white
self.window!.rootViewController = UIViewController()
self.window!.makeKeyAndVisible()
```

> info.plist中的`Main storyboard file base name`和`Launch screen interface file base name`与项目的`General`中的`Main Interface`和`Launch Screen File`是关联的。