## 1. Lifecyles
[Android-Lifecycle](https://www.jianshu.com/p/2c9bcbf092bc)
[Jetpack 之 LifeCycle 使用篇](https://juejin.im/post/5df64c19518825121d6e2013)

## 2. DataBinding
[Android DataBinding](https://www.jianshu.com/p/2c4ac24761f5)
[DataBinding](https://juejin.im/post/5d89d9f8f265da03f2340e2b)

## 3. Navigation
[Android Navigation库使用详解](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650243401&idx=1&sn=4a36c9904ae6dab73e5c3472b5ce4438&chksm=88637026bf14f930995d083df57c920eeda4fdd1a6ff63096b5338d4be23844a616d18f21e77&scene=38#wechat_redirect)
[Android官方架构组件Navigation](https://mp.weixin.qq.com/s?__biz=MzIwMTAzMTMxMg==&mid=2649492677&idx=1&sn=4d04f1045a01131d8bef185f625e2745&chksm=8eec873ab99b0e2cf8835f2a5b43f0c8aee4c7debb277895d850e8d12a3168a1cf2413d87161&scene=38#wechat_redirect)

## 4. ViewModel
[ViewModel由浅入深](https://www.jianshu.com/p/fdafc310fcdc)
[ViewModel由浅入深](https://juejin.im/post/5d5a3cd9f265da03c23ed40a)
[ViewModel的使用及原理解析](https://mp.weixin.qq.com/s/3Q3aJ0mYUWU5bo8rZOhDcA)
[剖析 Android 架构组件之 ViewModel](https://mp.weixin.qq.com/s?__biz=MzIxNzU1Nzk3OQ==&mid=2247487227&idx=1&sn=b38805ef59f9ecfeb5b72cbae2ed3df8&chksm=97f6b04fa081395970a81c480cf7c5dde3a02cc5cecb01da287189d59bd82dccd8de13418ffa&scene=38#wechat_redirect)

## 5. LiveData
LiveData 是一种可观察的数据存储器类。与常规的可观察类不同，LiveData 具有生命周期感知能力，意指它遵循其他应用组件（如 Activity、Fragment 或 Service）的生命周期。这种感知能力可确保 LiveData 仅更新处于活跃生命周期状态的应用组件观察者。通常在 ViewModel 内使用。

[深入理解架构组件：LiveData](https://github.com/googlesamples/android-sunflower)

可以重写 onActive 和 onInactive()，对应变为活跃和变为非活跃的时机，可执行一些资源、连接相关操作