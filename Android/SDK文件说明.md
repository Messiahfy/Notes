[官方链接](https://developer.android.google.cn/studio/intro/update.html#sdk-manager)
#文件说明
Android SDK Tools 必备。包含基本工具，例如 Android Emulator 和 ProGuard。位于SDK下的Tools目录
Android SDK Platform-Tools 必备。包含 Android 平台所需的各种工具，包括 adb 工具。位于SDK下的platform-tools目录
Android SDK Build-Tools 必备。包含构建 Android 应用的工具。位于SDK下的build-tools目录。Build-Tools可以安装一个或多个，因为某些工程用的旧版本构建工具。

Android Support Repository 推荐。包含支持库的本地 Maven 存储库，该存储库提供了一组丰富的 API，这些 API 兼容大多数版本的 Android。该工具是 Android Wear、Android TV 和 Google Cast 等产品的必备工具。位于SDK下的extras/android
主要是方便在gradle中使用android support libraries，因为Google并没有把这些库发布到maven center或者jcenter去，而是使用了Google自己的maven仓库。

在 SDK Platforms 选项卡中，您还必须安装至少一个版本的 Android 平台。
Android SDK Platform  必备。您的开发环境中必须至少有一个平台，您方可编译应用。为了在最新设备上提供最佳用户体验，请使用最新版本的平台作为构建目标。您的应用仍然可以在旧版系统上运行，但您必须以最新版本为目标构建应用，以便在安装最新版本 Android 的设备上运行应用时能够使用新功能。
即Android SDK Platform是向后（老版本）兼容的，开发时使用最新的版本来编译应用即可。位于SDK下的platforms目录

其他如System Image,Google某某库可忽略。
