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
Api 29以上，直接监听Activity回调；Api 29以下，通过Fragment生命周期


关键点2: 
LifecycleRegistry如何分发生命周期事件给LifecycleObserver

没有直接传递Lifecycle.Event，而是转为Lifecycle.State，跟原有State比较，再转为Event发给LifecycleObserver

目的在于可以确保收到完整的生命周期事件流程，以及防止自定义的情况生命周期状态不规范。


## ViewModel
* ViewModelProvider：获取ViewModel的壳子
* ViewModelProvider.Factory：创建ViewModel的工厂类
* ViewModelStoreOwner：提供ViewModelStore，例如Activity
* ViewModelStore：实际存储ViewModel的地方
通过ViewModelProvider获取ViewModel的核心逻辑就是，通过ViewModelStoreOwner的ViewModelStore获取ViewModel，如果不存在，就使用ViewModelProvider.Factory来创建ViewModel。

### ViewModel如何在配置变更重建情况保持？
配置变更时，ActivityThread调用`handleRelaunchActivity`重建Activity，会先调用`handleDestroyActivity`，其中会执行Activity的`retainNonConfigurationInstances()` --> `onRetainNonConfigurationInstance()`，
得到其中存储了`ViewModelStore`的`NonConfigurationInstances`对象。然后将此对象在执行`handleLaunchActivity` --> `performLaunchActivity`时传给新创建的Activity。

而对于Fragment，以androidx的1.4.0版本的Fragment为例，Fragment作为ViewModelStoreOwner的子类，实现的`getViewModelStore()`是从`FragmentManager`中获取`ViewModelStore`
```
@Override
public ViewModelStore getViewModelStore() {
    ...
    return mFragmentManager.getViewModelStore(this);
}
```
而`FragmentManager`的实现如下：
```
@NonNull
ViewModelStore getViewModelStore(@NonNull Fragment f) {
    return mNonConfig.getViewModelStore(f);
}
```
mNonConfig是FragmentManagerViewModel类型，是ViewModel的子类，每个Fragment的`ViewModelStore`就放在这里面。mNonConfig初始化如下：
```
if (parent != null) {
    mNonConfig = parent.mFragmentManager.getChildNonConfig(parent); //嵌套Fragment的情况
} else if (host instanceof ViewModelStoreOwner) {
    ViewModelStore viewModelStore = ((ViewModelStoreOwner) host).getViewModelStore();
    mNonConfig = FragmentManagerViewModel.getInstance(viewModelStore);
} else {
    mNonConfig = new FragmentManagerViewModel(false); //一般不走这个
}
```
以常见的情况，执行中间的分支为例，host为Activity
```
// FragmentManagerViewModel.getInstance方法
static FragmentManagerViewModel getInstance(ViewModelStore viewModelStore) {
    ViewModelProvider viewModelProvider = new ViewModelProvider(viewModelStore,
            FACTORY);
    return viewModelProvider.get(FragmentManagerViewModel.class);
}
```
也就是说，获取mNonConfig这个FragmentManagerViewModel类型的ViewModel，是从Activity的ViewModelStore中获取的。所以，**以Fragment为ViewModelStoreOwner获取ViewModel，就是把FragmentManagerViewModel存储于Activity的ViewModelStore中，Fragment作用域的ViewModel从FragmentManagerViewModel内提供的ViewModelStore中获取**。

### ViewModel有参构造函数
如果要保存状态到bundle中，需要使用SavedStateViewModelFactory（Activity和Fragment默认就是使用这个）。ViewModel的构造函数可以直接增加一个SavedStateHandle类型的参数，因为ComponentActivity的getDefaultViewModelProviderFactory()方法实现就是使用的SavedStateViewModelFactory，而SavedStateViewModelFactory的create方法就会使用反射先判断ViewModel的构造函数参数，如果有SavedStateHandle类型参数，就会帮我们传入Activity中得来的相关数据。Fragment同理。

如果构造函数要有自定义的参数，ViewModel的工厂可以继承AbstractSavedStateViewModelFactory

## LiveData
主动和被动使用数据 最佳用法？？？  是否最好应结合databinding？？？

LiveData 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。组件销毁时自动取消观察。通常在 ViewModel 内使用。

[深入理解架构组件：LiveData](https://github.com/googlesamples/android-sunflower)

如何做到仅在组件处于活跃生命周期时才更新数据？

观察LiveData时，会传入LifecycleOwner，只有其lifecycle状态为STARTED以上时，才会实际通知观察者。


如何做到组件回到活跃生命周期时立即更新数据？

观察LiveData传入的LifecycleOwner，一旦其生命周期状态变化到STARTED以上就会立即通知Observer

## DataBinding
数据绑定如果在xml中使用`ViewModel中的LiveData`或者是`androidx.databinding.Observable的子类`，编译时注解会生成对应的响应式监听更新代码

生成的binding类，设置数据，默认都会到下一帧才会实际刷新生效，如果要立即刷新生效，那么可以调用executePendingBindings()。如果设置LifecycleOwner，那么到生命周期STARTED的时候才生效

在databinding的xml文件中使用liveData或者BaseObservableField的子类型，在xml中会直接当成他们包装的原始类型使用，比如`LiveData<String>`会被当成String类型，当然，这只是在xml中使用的时候。