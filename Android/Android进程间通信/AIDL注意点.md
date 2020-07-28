AIDL中的定向 tag 表示了在跨进程通信中数据的流向，其中 in 表示数据只能由客户端流向服务端， out 表示数据只能由服务端流向客户端，而 inout 则表示数据可在服务端与客户端之间双向流通。其中，数据流向是针对在客户端中的那个传入方法的对象而言的。in 为定向 tag 的话表现为服务端将会接收到一个那个对象的完整数据，但是客户端的那个对象不会因为服务端对传参的修改而发生变动；out 的话表现为服务端将会接收到那个对象的参数为空的对象，但是在服务端对接收到的空对象有任何修改之后客户端将会同步变动；inout 为定向 tag 的情况下，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务端对该对象的任何变动。
[探索AIDL定向tag in out inout原理](https://www.jianshu.com/p/382633129b53)

[你真的理解AIDL中的in，out，inout么？](https://www.jianshu.com/p/ddbb40c7a251)


如果你对自定义类型使用了in或者inout标识符的话，你必须再给自定义类实现 readFromParcel()方法。比如：
public void readFromParcel(Parcel in) {
  name = in.readString();
  skill = in.readString();
}

1. AIDL方法在服务端的Binder线程池中执行，因此多个客户端同时连接时，会存在多个线程同时访问的情形，所以我们要在AIDL方法（Stub的实现）中处理线程同步。
2. AIDL中不能使用普通java接口，而是aidl接口，且回调时在Binder线程池中执行，如果要进行UI操作，则要切换线程。
3. 因为跨进程，所以传到服务端的接口是一个新的对象，所以需要使用RemoteCallbackList来解注册。
4. 客户端的onServiceConnected和onServiceDisconnected方法都运行在UI线程中，所以不可以在其中直接调用服务端的耗时方法；服务端的方法本身就运行在服务端的Binder线程池中，所以本身就可以执行大量耗时操作，而不用开线程。
5. Binder的linkToDeath和unlinkToDeath方法，设置DeathRecipient 接口，会在客户端的Binder线程池中被回调，而onServiceDisconnected在客户端的UI线程被回调，也就是说DeathRecipient 的回调不能访问UI。
6. AIDL权限验证，可以使用Service的onBind中验证，不通过就返回null；也可以在服务端的onTransact中返回false。