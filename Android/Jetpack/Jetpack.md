# Jetpack概述
现有的 support library 和架构组件组合起来，现在统一放到androidX库中，也就是说把提高Android开发体验的一系列库做了统一。

https://developer.android.google.cn/jetpack/androidx/releases/lifecycle

## Lifecyles

提供了一套类和接口，可用于构建能感知生命周期的组件。
好处就是可以将生命周期回调中的代码移到组件本身中。

LifecycleOwner（Activity/Fragment等）提供一个 Lifecycle 对象

LifecycleObserver 就对这个 Lifecycle（实现类为LifecycleRegistry） 对象进行观察

Lifecycle 作为中转站来传递生命周期事件到LifecycleObserver

LifecycleRegistry如何接收到LifecycleOwner的生命周期回调

使用无界面Fragment
Api 29以上，直接监听Activity回调Api 29以下，通过Fragment生命周期


关键点2: 
LifecycleRegistry如何分发生命周期事件给LifecycleObserver

没有直接传递Lifecycle.Event，而是转为Lifecycle.State，跟原有State比较，再转为Event发给LifecycleObserver

目的在于可以确保收到完整的生命周期事件流程，以及防止自定义的情况生命周期状态不规范。


## ViewModel
ViewModelProvider 获取ViewModel的壳子

ViewModelStoreOwner（Activity等） 提供ViewModelStore 

ViewModelStore 是实际存储ViewModel的地方


ViewModel如何在配置变更重建情况保持？
 
配置变更时，ActivityThread调用handleRelaunchActivity重建Activity，会先调用handleDestroyActivity，其中会执行Activity的retainNonConfigurationInstances()-->onRetainNonConfigurationInstance()，
得到其中存储了ViewModelStore的NonConfigurationInstances对象。

然后将此对象在执行handleLaunchActivity-->performLaunchActivity 时传给新创建的Activity

## LiveData
LiveData 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。组件销毁时自动取消观察。通常在 ViewModel 内使用。

[深入理解架构组件：LiveData](https://github.com/googlesamples/android-sunflower)

如何做到仅在组件处于活跃生命周期时才更新数据？

观察LiveData时，会传入LifecycleOwner，只有其lifecycle状态为STARTED以上时，才会实际通知观察者。


如何做到组件回到活跃生命周期时立即更新数据？

观察LiveData传入的LifecycleOwner，一旦其生命周期状态变化到STARTED以上就会立即通知Observer
