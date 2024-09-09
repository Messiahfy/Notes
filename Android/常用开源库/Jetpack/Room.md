## Room的响应式
* LiveData：查询函数返回LiveData类型。源码对应InvalidationTracker#createLiveData
* kotlin协程：查询函数返回Flow。源码对应CoroutinesRoom#createFlow
* RxJava

@Query 方法：Room 支持 Publisher、Flowable 和 Observable 类型的返回值。
@Insert、@Update 和 @Delete 方法：Room 2.1.0 及更高版本支持 Completable、Single<T> 和 Maybe<T> 类型的返回值。

> Room的编译时注解处理代码中，会对Kotlin协程、RxJava之类的返回类型做支持，而且还支持了Paging库的DataSource.Factory（Paging2）、PagingSource（Paging3），所以在使用Room的时候，可以自己到[源码](https://androidx.tech/artifacts/room/room-compiler/2.3.0-source/androidx/room/ext/javapoet_ext.kt.html)中去了解还支持了哪些类型，文档中可能并不一定都详细说明。

## OnConflictStrategy
在使用@Insert插入数据时，如果表中已经包含此主键数据，Room将根据我们的冲突策略处理：
* NONE：和ABORT一致
* REPLACE：如果发生冲突，使用插入的数据覆盖已存在的数据
* ABORT：回滚当前事务，并抛出异常SQLiteConstraintException
* IGNORE：如果发生冲突，将忽略此次调用，并返回-1

## 
线程池？transactionExecutor  queryExecutor  queryCallbackExecutor