[Mockito框架的使用](https://blog.csdn.net/qq_17766199/article/details/78450007)

## 1. Mockito概述
&emsp;&emsp;Mockito是Java中的一个Mock框架。mock对象就是在调试期间用来作为真实对象的替代品。mock测试就是在测试过程中，对那些不容易构建的对象用一个虚拟对象来代替测试的方法就叫mock测试。什么要使用mock呢？关键就在于专注于测试本类，可以对于依赖的对象可以使用mock来得到，做到依赖隔离。

&emsp;&emsp;mock可以达到两个目的：
1. 验证此类的某些方法的调用执行情况，比如调用次数和参数是什么
2. 方便指定修改方法的行为，比如指定返回值

## PowerMock框架
拓展了Mockito框架，从而支持了mock static方法、private方法、final方法与类等

## Kotlin有对应的 mockK 库