![Android系统Window层级](https://upload-images.jianshu.io/upload_images/3468445-467e8bdf84cca4ea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


PopupWindow没有创建Window，只是创建了新的的DecorView

Dialog用的单独的PhoneWindow，为应用程序Window，也是另外的DecorView

PopupWindow和Dialog的配置方法大都是public，所以自定义PopupWindow和Dialog既可以在子类的构造方法或者onCreate等中调用setContentView()等配置方法，也可以直接构造PopupWindow和Dialog实例，再用实例调用配置方法。
