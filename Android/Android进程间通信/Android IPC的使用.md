Android常用跨进程通信的方式：
1.intent/bundle 单向通信，无法同步调用
2.文件
3.messenger（基于AIDL，归根到底还是Binder）
4.AIDL（基于Binder）
5.ContentProvider（基于Binder）
6.Socket

# 1. 开启多进程
&emsp;&emsp;给Android的四大组件在manifest文件中指定`android:process`属性，除此之外可以通过JNI在native层去fork一个进程，但这种方式不常用。  
&#160; &#160; &#160; &#160;例如为activity指定进程：
```
<activity android:name=".MainActivity">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />
            <category android:name="android.intent.category.LAUNCHER" />
         </intent-filter>
</activity>
<activity
        android:name=".SecondActivity"
        android:process=":remote" />
<activity
        android:name=".ThirdActivity"
        android:process="com.hfy.example.remote" />
```
&#160; &#160; &#160; &#160;MainActivity在应用包名默认进程；SecondActivity的写法是简写，实际是在包名后加上:remote；ThirdActivity直接指定了进程名称。  
&#160; &#160; &#160; &#160;`:冒号`的这种写法代表该进程为当前应用的私有进程，其他应用的组件不能和它在同一进程。
![指定进程后的三个进程.png](https://upload-images.jianshu.io/upload_images/3468445-f04ef9333a5308e2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)、
# 2. 多进程模式产生的问题
* 静态成员和单例模式失效
* 线程同步机制失效
* SharedPreferences的可靠性下降
* Application会多次创建
# 3. 进程间通信的基础概念
## 3.1 Serializable
java序列化对象方式
## 3.2 Parcelable
`Android`特有序列化对象方式，`Parcel`中包装了可序列化的数据。
```
public class Book implements Parcelable {
    public int bookId;
    public String bookName;

    protected Book(Parcel in) {//从序列化后的对象中创建原始对象
        bookId = in.readInt();
        bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {//从序列化后的对象中创建原始对象
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];//创建指定长度的原始对象数组
        }
    };

    @Override
    public int describeContents() {//返回当前对象的内容描述。如果含有文件描述符则返回1，否则返回0，几乎都返回0
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {//将当前对象写入序列化结构中
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }
}
```
&#160; &#160; &#160; &#160;系统中如`Intent`、`Bundle`、`Bitmap`都是实现了`Parcelable`接口的，`List`和`Map`当其中的每个元素都是可序列化的则也可以序列化。
# 4. 选用AIDL生成的类来分析Binder的工作机制
# 4.1 创建文件
![目录](https://upload-images.jianshu.io/upload_images/3468445-e374408d8547190c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

&#160; &#160; &#160; &#160;创建`Book.java`，`Book.aidl`，`IBookManager.aidl`，其中`Book.java`和`Book.aidl`所在的包**名**必须一致（都放到main/aidl/...的包下也可以，但是这样就需要在在build.gradle文件 android{ } 的sourceSet中增加上aidl的目录）。
&#160; &#160; &#160; &#160;`Book.java`与上文一致，`Book.aidl`和`IBookManager.aidl`如下：
```
// Book.aidl
package com.example.hfy.test.aidl;

parcelable Book;
```
```
// IBookManager.aidl
package com.example.hfy.test.aidl;

import com.example.hfy.test.aidl.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```
&#160; &#160; &#160; &#160;`Book.java`是表示图书信息的类，实现了`Parcelable`接口。`Book.aidl`是Book类在`AIDL` 中的声明。`IBookManager.aidl`是定义的一个接口，包含两个方法。尽管`Book`类和`IBookManager.aidl`在同一包中也要导入`Book`类，这是`aidl`的特殊之处。系统会通过`IBookManager.aidl`在`generated/source/aidl/com.example.hfy.test.aidl`目录中生成一个`IBookManager.java`类，根据这个类来分析`Binder`的工作原理。
# 4.2 分析通过AIDL生成的java类
```
package com.example.hfy.test.aidl;

public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.hfy.test.aidl.IBookManager {
        //Binder的唯一标识
        private static final java.lang.String DESCRIPTOR = "com.example.hfy.test.aidl.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * 将服务端的Binder对象转换为客户端所需的AIDL接口类型的对象，如果在同一进
         * 程就返回服务端的Stub本身，否则返回Stub.Proxy
         */
        public static com.example.hfy.test.aidl.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.hfy.test.aidl.IBookManager))) {
                return ((com.example.hfy.test.aidl.IBookManager) iin);
            }
            return new com.example.hfy.test.aidl.IBookManager.Stub.Proxy(obj);
        }
        //返回当前Binder对象
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }
        /**
        *这个方法运行在服务端的Binder线程池中，当客户端发起跨进程请求时，远程请求 
        *会通过系统底层封装后交由此方法处理。通过code确定客户端请求的目标方法是什 
        *么，接着从data中取出目标方法所需的参数（如果有的话），然后执行目标方法。 
        *当方法执行完毕后，就向reply中写入返回值（如果有的话），onTransact方法的过 
        *程就是这样的。如果返回false，那么客户端的请求会失败，可以用这个特性来做权 
        *限验证。
        */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<com.example.hfy.test.aidl.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    com.example.hfy.test.aidl.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.hfy.test.aidl.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.example.hfy.test.aidl.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /** 此方法运行在客户端，当客户端远程调用此方法时，内部实现是这样的：首先 
            * 创建该方法所需要的输入型Parcel对象_data、输出型对象_reply，和返回值对
            * 象_result；然后把该方法的参数信息写入_data中（如果有参数的话）；接着调
            * 用transact方法来发起RPC（远程过程调用）请求，同时当前线程挂起；然后服
            * 务端的onTransact方法会被调用，直到RPC返回后，当前线程继续执行，并从 
            * _reply中取出RPC过程的返回结果；最后返回_reply中的数据。
            * （发送远程请求，当前线程会被挂起直至服务端返回数据，所以如果耗时则不能 
            * 在UI线程发起远程请求；其次，由于服务端的Binder方法运行在Binder的线程池 
            * 中，所以Binder方法不管是否耗时都应该用同步方式实现，因为它已经运行在一 
            * 个线程中）
            */
            @Override
            public java.util.List<com.example.hfy.test.aidl.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.hfy.test.aidl.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.hfy.test.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.example.hfy.test.aidl.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public java.util.List<com.example.hfy.test.aidl.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.example.hfy.test.aidl.Book book) throws android.os.RemoteException;
}
```
&#160; &#160; &#160; &#160;`IBookManager.java`继承了`IInterface`接口，并且自身也是一个接口，它声明了在`IBookManager.aidl`中声明的两个方法。且声明了一个内部类`Stub`，这个`Stub`是一个`Binder`类，`Stub`中声明了一个静态内部类`Proxy`来代理Binder的`跨进程调用`。
&#160; &#160; &#160; &#160;客户端在`onServiceConnected`中拿到的`IBookManager `实际是`Proxy`（IBookManager.Stub.asInterface(IBinder)），`IBookManager` 的`getBookList()`方法即`Proxy`的`getBookList()`方法运行在客户端，而实际重写的Stub的getBookList()在Proxy调用后在`onTransact()`中被调用，onTransact()发生在服务端的`Binder线程池`。
## 4.3 Binder（Stub）工作机制图如下：![Binder工作机制图（来自Android开发艺术探索）](https://upload-images.jianshu.io/upload_images/3468445-1800e0150d307af3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.4 Binder连接中断的几种处理方式
*  远程调用，捕获DeadObjectException（RemoteException的子类）异常，它们是在连接中断时引发的；这将是远程方法引发的唯一异常。
* Binder的isBinderAlive()
* Binder的pingBinder()
* Binder的linkToDeath和unlinkToDeath方法，设置DeathRecipient 接口