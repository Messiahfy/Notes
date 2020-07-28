https://developer.android.google.cn/training/dependency-injection/dagger-android#kotlin
https://codelabs.developers.google.com/codelabs/android-dagger/#6
https://dagger.dev/dev-guide/android
https://blog.mindorks.com/the-new-dagger-2-android-injector-cbe7d55afa6a
https://www.jianshu.com/p/8060a260488d

管理对象和它的生命周期

Module提供对象，通过Component注入目标类

@Singleton是局部单例，仅在同一个Component实例范围内是单例，在生成的DaggerXxxComponent中，会用同一个DoubleCheck来获取对象

要实现全局单例，就要用同一个Component，比如在Application中初始化Component，然后都用这个Component

Component中的方法，带参数的方法就是用来注入依赖到参数类型的对象，带返回值的方法用于提供依赖给父Component

如果不能用@Inject注解构造函数，比如该类是第三方库，那么可以使用@Provides和@Binds来注解另外的方法