`DataStore`的`edit`方法（或者`updateData`方法）符合原子事务性和有序性：

1. 原子事务性：
`DataStore`的实现类`SingleProcessDataStore`中可以看到，写入数据最终调用的`writeData`方法会先把数据写入临时文件，并且调用fd.sync()，之后再重命名为最终文件名。这样就保证了写入数据的原子事物性。

2. 有序性：
```
lifecycleScope.launch {
    dataStore.edit {
        delay(2000)
        it[KEY_A] = "123"
    }
}

lifecycleScope.launch {
    delay(1000)
    dataStore.edit {
        it[KEY_A] = "456"
    }
}
```
在上面的例子中，在主线程启动了两个协程，第一个协程调用`dataStore.edit`方法，但执行时会延时2秒模拟耗时操作，第二个协程先延时1秒再调用`dataStore.edit`方法，虽然第二个协程只延时1秒就调用了`dataStore.edit`方法，但由于DataStore的有序性设计（内部的SimpleActor类来保证有序性），第一个`dataStore.edit`执行代码块已经在执行，虽然耗时2秒，但第二个`dataStore.edit`执行代码块仍然需要等待第一个`dataStore.edit`代码块执行完成后再执行，所以数据会先改为“123”然后很快改为“456”。

写入数据后会更新`downstreamFlow`（MutableStateFlow类型），`DataStore`读取数据时使用的`data`（Flow类型）就是从`downstreamFlow`转换而来。