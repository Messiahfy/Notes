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


### ViewModel的移除
* Activity：ComponentActivity会调用getLifecycle()注册生命周期Observer，在onDestroy时，判断isChangingConfigurations()为false，才会移除ViewModelStore中的ViewModel实例。
* Fragment：调用FragmentStateManager destroy，根据情况，如果不是配置变更，而是正常的销毁Fragment，就会移除ViewModel并调用onCleared，然后执行onDestory。所以如果Fragment中使用ViewModel是懒加载，并且在onDestory前没有调用该ViewModel，会导致在onDestory时才实例化ViewModel，而移除ViewModel在onDestory之前，也就无法执行onCleared，所以在特殊情况下，ViewModel的init和onCleared可能不是对称调用。

### ViewModel有参构造函数
如果要保存状态到bundle中，需要使用SavedStateViewModelFactory（Activity和Fragment默认就是使用这个）。ViewModel的构造函数可以直接增加一个SavedStateHandle类型的参数，因为ComponentActivity的getDefaultViewModelProviderFactory()方法实现就是使用的SavedStateViewModelFactory，而SavedStateViewModelFactory的create方法就会使用反射先判断ViewModel的构造函数参数，如果有SavedStateHandle类型参数，就会帮我们传入Activity中得来的相关数据。Fragment同理。

如果构造函数要有自定义的参数，ViewModel的工厂可以继承AbstractSavedStateViewModelFactory


### ViewModel构造流程核心源码

想要构造ViewModel时，会先构造 ViewModelProvider，然后调用get函数。通过分析核心源码了解ViewModel构造流程，以及官方如何支持`SavedStateHandle`的使用。

先了解 ViewModelProvider 的构造方式：
* `ViewModelProvider`有多个重载方法，没有传入自定义的 Factory 和 CreationExtras，则会使用默认的 ViewModelProvider.Factory 和 CreationExtras
* 在使用`androidx`的ComponentActivity和Fragment时，默认的 Factory 和 CreationExtras 就是实现`HasDefaultViewModelProviderFactory`接口的返回值。如果没有实现接口，则为`NewInstanceFactory`和`CreationExtras.Empty`

```
public open class ViewModelProvider

@JvmOverloads
constructor(
    private val store: ViewModelStore,
    private val factory: Factory,
    private val defaultCreationExtras: CreationExtras = CreationExtras.Empty,
) {
    // 重载构造函数。
    // defaultFactory(owner)会判断owner实现HasDefaultViewModelProviderFactory接口则获取接口返回的Factory，否则使用默认的NewInstanceFactory
    // defaultCreationExtras(owner)会判断owner实现HasDefaultViewModelProviderFactory则获取返回的CreationExtras，否则为默认值CreationExtras.Empty
    // androidx的ComponentActivity和Fragment都实现了这些接口

    public constructor(
        owner: ViewModelStoreOwner
    ) : this(owner.viewModelStore, defaultFactory(owner), defaultCreationExtras(owner))

    // 重载构造函数。
    public constructor(owner: ViewModelStoreOwner, factory: Factory) : this(
        owner.viewModelStore,
        factory,
        defaultCreationExtras(owner)
    )


    // ViewModelProvider.Factory接口包含两个函数，用于创建ViewModel
    public interface Factory {
        
        public fun <T : ViewModel> create(modelClass: Class<T>): T {
            throw UnsupportedOperationException(
                "Factory.create(String) is unsupported.  This Factory requires " +
                    "`CreationExtras` to be passed into `create` method."
            )
        }

        // 这个重载函数，多一个 CreationExtras 类型的参数，ViewModelProvider构造函数传入的CreationExtras就会传到这里
        public fun <T : ViewModel> create(modelClass: Class<T>, extras: CreationExtras): T =
            create(modelClass)
    }


    // ViewModelProvider get函数实际会调用重载函数
    @MainThread
    public open operator fun <T : ViewModel> get(modelClass: Class<T>): T {
        val canonicalName = modelClass.canonicalName
            ?: throw IllegalArgumentException("Local and anonymous classes can not be ViewModels")
        return get("$DEFAULT_KEY:$canonicalName", modelClass)
    }

    // ViewModelProvider 最终会调用这个get函数
    @MainThread
    public open operator fun <T : ViewModel> get(key: String, modelClass: Class<T>): T {
        val viewModel = store[key]

        // ViewModel有缓存的情况

        if (modelClass.isInstance(viewModel)) {
            (factory as? OnRequeryFactory)?.onRequery(viewModel!!)
            return viewModel as T
        } else {
            @Suppress("ControlFlowWithEmptyBody")
            if (viewModel != null) {
                // TODO: log a warning.
            }
        }
        // ViewModelProvider构造函数传入的defaultCreationExtras，会传给Factory

        val extras = MutableCreationExtras(defaultCreationExtras)
        extras[VIEW_MODEL_KEY] = key // 将key也放到CreationExtras中，Factory中可能会使用
        
        // 调用ViewModelProvider.Factory的create函数，优先调用有CreationExtras参数的

        return try {
            factory.create(modelClass, extras)
        } catch (e: AbstractMethodError) {
            factory.create(modelClass)
        }.also { store.put(key, it) }
    }

}
```
`ViewModelProvider`调用get()方法时，实际就会调用`Factory`的create函数，并把`CreationExtras`传给`Factory`。


`ComponentActivity`中的`HasDefaultViewModelProviderFactory`默认实现
```
//  ComponentActivity 类

class ComponentActivity extends androidx.core.app.ComponentActivity implements
        LifecycleOwner,
        ViewModelStoreOwner,
        HasDefaultViewModelProviderFactory,
        SavedStateRegistryOwner,
        ...... // 省略其他接口
{

    // 默认的 Factory 为 SavedStateViewModelFactory
    override val defaultViewModelProviderFactory: ViewModelProvider.Factory by lazy {
        SavedStateViewModelFactory(
            application,
            this,
            if (intent != null) intent.extras else null
        )
    }

    // 默认的 CreationExtras 存储了如下对象
    @get:CallSuper
    override val defaultViewModelCreationExtras: CreationExtras

        get() {
            val extras = MutableCreationExtras()
            if (application != null) {
                extras[APPLICATION_KEY] = application  // 存储Application
            }
            extras[SAVED_STATE_REGISTRY_OWNER_KEY] = this  // 存储 SavedStateRegistryOwner 对象，即Activity自身
            extras[VIEW_MODEL_STORE_OWNER_KEY] = this  // 存储 ViewModelStoreOwner 对象，即Activity自身
            val intentExtras = intent?.extras
            if (intentExtras != null) {
                extras[DEFAULT_ARGS_KEY] = intentExtras  // 存储 Intent 中的Bundle
            }
            return extras
        }
}
```

AndroidX的`Fragment`中的`HasDefaultViewModelProviderFactory`默认实现
```
class Fragment implements ComponentCallbacks, OnCreateContextMenuListener, LifecycleOwner,
        ViewModelStoreOwner, HasDefaultViewModelProviderFactory, SavedStateRegistryOwner,
        ActivityResultCaller {

    // 默认使用 SavedStateViewModelFactory
    public ViewModelProvider.Factory getDefaultViewModelProviderFactory() {
        // ......
        if (mDefaultFactory == null) {
            // ...... 省略获取application的代码
            mDefaultFactory = new SavedStateViewModelFactory(
                    application,
                    this,
                    getArguments());
        }
        return mDefaultFactory;
    }

    // 默认的 CreationExtras
    @NonNull
    @Override
    @CallSuper
    public CreationExtras getDefaultViewModelCreationExtras() {
        // ...... 省略获取application的代码
        MutableCreationExtras extras = new MutableCreationExtras();
        if (application != null) {
            // 存储Application
            extras.set(ViewModelProvider.AndroidViewModelFactory.APPLICATION_KEY, application);
        }
        // 存储 SavedStateRegistryOwner 对象，即 Fragment 自身

        extras.set(SavedStateHandleSupport.SAVED_STATE_REGISTRY_OWNER_KEY, this);
        // 存储 ViewModelStoreOwner 对象，即 Fragment 自身

        extras.set(SavedStateHandleSupport.VIEW_MODEL_STORE_OWNER_KEY, this);
        if (getArguments() != null) {
            // 存储 Bundle 对象，即 getArguments()

            extras.set(SavedStateHandleSupport.DEFAULT_ARGS_KEY, getArguments());
        }
        return extras;
    }
}
```

前面了解了`ViewModelPorvider`的构造方式、`Factory`和`CreationExtras`默认值的获取，以及传递给Factory的情况。下面来看Factory如何构造ViewModel。


默认使用的是`SavedStateViewModelFactory`类型：

AndroidX的`ComponentActivity`和`Fragment`在构造`SavedStateViewModelFactory`时，传的 defaultArgs 就分别是Intent.extras 和 getArguments()，同时在`CreationExtras`中key为`DEFAULT_ARGS_KEY`的值也设置了Intent.extras 和 getArguments()，所以在`SavedStateViewModelFactory`中可以通过这两个地方获取到`Bundle`参数。
```
class SavedStateViewModelFactory : ViewModelProvider.OnRequeryFactory, ViewModelProvider.Factory {
    // .....

    constructor(application: Application?, owner: SavedStateRegistryOwner, defaultArgs: Bundle?) {
        savedStateRegistry = owner.savedStateRegistry
        lifecycle = owner.lifecycle
        this.defaultArgs = defaultArgs
        this.application = application
        factory = if (application != null) getInstance(application)
            else ViewModelProvider.AndroidViewModelFactory()
    }


    override fun <T : ViewModel> create(modelClass: Class<T>, extras: CreationExtras): T {
        val key = extras[ViewModelProvider.NewInstanceFactory.VIEW_MODEL_KEY]
            ?: throw IllegalStateException(
                "VIEW_MODEL_KEY must always be provided by ViewModelProvider"
            )

        return if (extras[SAVED_STATE_REGISTRY_OWNER_KEY] != null &&
            extras[VIEW_MODEL_STORE_OWNER_KEY] != null) {
                // CreationExtras 中存储了 SavedStateRegistryOwner 和 ViewModelStoreOwner 对象的情况，
                // AndroidX的`ComponentActivity`和`Fragment`默认已经设置了

            // 从 CreationExtras 获取 Application

            val application = extras[ViewModelProvider.AndroidViewModelFactory.APPLICATION_KEY]
            val isAndroidViewModel = AndroidViewModel::class.java.isAssignableFrom(modelClass)
            val constructor: Constructor<T>? = if (isAndroidViewModel && application != null) {
                // 如果是 AndroidViewModel 的子类，得到构造函数（包含SavedStateHandle参数的情况）

                findMatchingConstructor(modelClass, ANDROID_VIEWMODEL_SIGNATURE)
            } else {
                // 其他 ViewModel 类型，得到仅包含SavedStateHandle参数

                findMatchingConstructor(modelClass, VIEWMODEL_SIGNATURE)
            }

            // 找不到构造函数，即不需要SavedStateHandle参数
            if (constructor == null) {
                // 则直接使用 AndroidViewModelFactory
                return factory.create(modelClass, extras)
            }

            // 如果找到了包含SavedStateHandle参数的构造函数，则将 CreationExtras 转换为 SavedStateHandle传给ViewModel构造函数

            // CreationExtras 内的DEFAULT_ARGS_KEY对应的Bundle的key-value将会存到 SavedStateHandle 中
            val viewModel = if (isAndroidViewModel && application != null) {
                newInstance(modelClass, constructor, application, extras.createSavedStateHandle())
            } else {
                newInstance(modelClass, constructor, extras.createSavedStateHandle())
            }
            viewModel
        } else {

            // 如果CreationExtras 中没有存储 SavedStateRegistryOwner 和 ViewModelStoreOwner 对象，
            // 则调用重载函数，内部会通过 LegacySavedStateHandleController 来构造 SavedStateHandle

            val viewModel = if (lifecycle != null) {
                create(key, modelClass)
            } else {
                throw IllegalStateException("SAVED_STATE_REGISTRY_OWNER_KEY and" +
                    "VIEW_MODEL_STORE_OWNER_KEY must be provided in the creation extras to" +
                    "successfully create a ViewModel.")
            }
            viewModel
        }
    }
}
```
如果是自定义`AbstractSavedStateViewModelFactory`子类，内部也是通过`LegacySavedStateHandleController`来构造`SavedStateHandle`，需要自己构造子类时传值Bundle或者调用ViewModelProvier的get时传值Bundle到`CreationExtras`的`DEFAULT_ARGS_KEY`对应值中。

> 上面以`SavedStateViewModelFactory`为例，其他类型的 ViewModelFactory 则由具体的实现决定。


### SavedStateHandle的重建场景

#### ComponentActivity的情况
androidx.activity 1.9.0版本

1. init 初始化时，执行`enableSavedStateHandles()`，会把`SavedStateHandlesProvider`注册到`SavedStateRegistry`中。
2. （场景A）构造ViewModel时（默认使用`SavedStateViewModelFactory`的情况），调用`createSavedStateHandle`构造`SavedStatehandle`，会把`SavedStatehandle`存到`viewModelStoreOwner.savedStateHandlesVM.handles`中。
3. （场景B）如果使用`AbstractSavedStateViewModelFactory`子类的情况，`LegacySavedStateHandleController.create()`返回`SavedStateHandleController`调用`attachToLifecycle`方法会调用`SavedStateRegistry`的`registerSavedStateProvider`方法，注册了`SavedStateHandle`的`savedStateProvider`对象。
4. `ComponentActivity`在`onSaveInstanceState`时会调用`SavedStateRegistry.performSave`，从而调用注册的`SavedStateHandlesProvider`的`saveState()`方法，**场景A**会从`viewModelStoreOwner.savedStateHandlesVM.handles中`遍历`SavedStatehandle`，调用它们的`savedStateProvider.saveState()`，**场景B**则直接调用`SavedStateHandle`的`savedStateProvider`对象的saveState()方法。这样就把数据保存到了`onSaveInstanceState`的Bundle中，数据的key为`SAVED_COMPONENTS_KEY`。
5. onCreate时，调用`savedStateRegistryController.performRestore`--`savedStateRegistry.performRestore`，会把 key为`SAVED_COMPONENTS_KEY`的数据拿出来放到`restoredState`字段中。`场景A`和`场景B`在创建`SavedStateHandle`时，都会取出恢复的数据，并写入创建的`SavedStateHandle`中。


#### Fragment的情况
androidx 1.6.2版本

1. Fragment注册了OnPreAttachListener，在`performAttach()`时就会调用`enableSavedStateHandles`，和Activity一样把`SavedStateHandlesProvider`注册到`SavedStateRegistry`中
2. 构造ViewModel，和Activity类型，忽略调用流程。
3. `FragmentStateManager.saveState()`时，会调用`mFragment.mSavedStateRegistryController.performSave`，则会执行每个`SavedStateHandle`的`savedStateProvider`对象的saveState()方法。
4. 注册的OnPreAttachListener回调，也会调用`mSavedStateRegistryController.performRestore`--`savedStateRegistry.performRestore`，那么也是在创建`SavedStateHandle`时拿到恢复的数据并写入。

> 使用Hilt的情况，默认是`HiltViewModelFactory`，它是`AbstractSavedStateViewModelFactory`的子类。

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