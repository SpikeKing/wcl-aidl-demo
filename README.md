# 使用 AIDL 实现 Android 的跨进程通信

> 欢迎Follow我的[GitHub](https://github.com/SpikeKing), 关注我的[简书](http://www.jianshu.com/users/e2b4dd6d3eb4/). 其余参考[Android目录](http://www.jianshu.com/p/780658b79227).

**AIDL(Android Interface Definition Language)**, 即Android接口定义语言. 在Android中, AIDL是跨进程通信的主要实现方式. 我们同样也可以使用AIDL, 实现自己的跨进程方案. 本文介绍AIDL的使用方式.

服务端: 创建Service服务监听客户端的请求, 实现AIDL接口.

客户端: 绑定服务端, 调用AIDL的方法.

AIDL接口: 跨进程通信的接口, AIDL的包名需要与项目的包名相同, 默认生成即可.

> AIDL支持的数据类型: 基本类型, 字符串类型(String&CharSequence), List, Map, Parcelable, AIDL接口. 共六种.

**流程**: 客户端注册服务端, 服务端添加新书, 客户端接收, 并提供客户端的查询书数量的接口.

本文源码的GitHub[下载地址](https://github.com/SpikeKing/wcl-aidl-demo)

---

## AIDL

本文使用自定义的数据类型**Book**类, 实现Parcelable接口, 具体[参考](http://www.jianshu.com/p/496646bc1f25).

``` java
public class Book implements Parcelable {

    public int bookId;
    public String bookName;

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    @Override public int describeContents() {
        return 0;
    }

    @Override public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }

    public static final Parcelable.Creator<Book> CREATOR = new Parcelable.Creator<Book>() {
        @Override public Book createFromParcel(Parcel source) {
            return new Book(source);
        }

        @Override public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    private Book(Parcel source) {
        bookId = source.readInt();
        bookName = source.readString();
    }

    @Override public String toString() {
        return "ID: " + bookId + ", BookName: " + bookName;
    }
}
```

AIDL使用自定义类, 需要声明Parcelable类.

``` java
// IBook.aidl
package org.wangchenlong.wcl_aidl_demo;

// Declare any non-default types here with import statements

parcelable Book;
```

添加AIDL的接口, 用于通知新书到达.

``` java
// IOnNewBookArrivedListener.aidl
package org.wangchenlong.wcl_aidl_demo;

// Declare any non-default types here with import statements
import org.wangchenlong.wcl_aidl_demo.Book;

interface IOnNewBookArrivedListener {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
//    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
//            double aDouble, String aString);

    void onNewBookArrived(in Book newBook);
}

```

> AIDL文件注释较多, 都是自动生成, 不影响阅读.

核心AIDL类, 书籍管理器, 四个方法, 获取图书列表, 添加书籍, 注册接口, 解注册接口. 注意, 使用其他方法, 需要**import**导入相应文件.

``` java
// IBookManager.aidl
package org.wangchenlong.wcl_aidl_demo;

// Declare any non-default types here with import statements

import org.wangchenlong.wcl_aidl_demo.Book;
import org.wangchenlong.wcl_aidl_demo.IOnNewBookArrivedListener;

interface IBookManager {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
//    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
//            double aDouble, String aString);
    List<Book> getBookList(); // 返回书籍列表
    void addBook(in Book book); // 添加书籍
    void registerListener(IOnNewBookArrivedListener listener); // 注册接口
    void unregisterListener(IOnNewBookArrivedListener listener); // 注册接口
}

```

> 所有的参数都需要标注参数方向, in表示输入类型, out表示输出类型, inout表示输入输出类型. out与inout的开销较大, 不能统一使用高级方向.

---

## 服务端

服务端通过**Binder**实现AIDL的``IBookManager.Stub``接口.

``` java
private Binder mBinder = new IBookManager.Stub() {
    @Override public List<Book> getBookList() throws RemoteException {
        SystemClock.sleep(5000); // 延迟加载
        return mBookList;
    }

    @Override public void addBook(Book book) throws RemoteException {
        mBookList.add(book);
    }

    @Override
    public void registerListener(IOnNewBookArrivedListener listener) throws RemoteException {
        mListenerList.register(listener);
        int num = mListenerList.beginBroadcast();
        mListenerList.finishBroadcast();
        Log.e(TAG, "添加完成, 注册接口数: " + num);
    }

    @Override
    public void unregisterListener(IOnNewBookArrivedListener listener) throws RemoteException {
        mListenerList.unregister(listener);
        int num = mListenerList.beginBroadcast();
        mListenerList.finishBroadcast();
        Log.e(TAG, "删除完成, 注册接口数: " + num);
    }
};
```

服务启动时, 添加两本新书, 并使用线程继续添加.

``` java
@Override public void onCreate() {
    super.onCreate();
    mBookList.add(new Book(1, "Android"));
    mBookList.add(new Book(2, "iOS"));
    new Thread(new ServiceWorker()).start();
}
```

添加书籍, 并发送通知.

``` java
private class ServiceWorker implements Runnable {
    @Override public void run() {
        while (!mIsServiceDestroyed.get()) {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            num++;
            if (num == 5) {
                mIsServiceDestroyed.set(true);
            }
            Message msg = new Message();
            mHandler.sendMessage(msg); // 向Handler发送消息,更新UI
        }
    }
}

private Handler mHandler = new Handler() {
    public void handleMessage(Message msg) {
        int bookId = 1 + mBookList.size();
        Book newBook = new Book(bookId, "新书#" + bookId);
        try {
            onNewBookArrived(newBook);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
};
```

向注册监听的, 发送新书的添加通知.

``` java
private void onNewBookArrived(Book book) throws RemoteException {
    mBookList.add(book);
    Log.e(TAG, "发送通知的数量: " + mBookList.size());
    int num = mListenerList.beginBroadcast();
    for (int i = 0; i < num; ++i) {
        IOnNewBookArrivedListener listener = mListenerList.getBroadcastItem(i);
        Log.e(TAG, "发送通知: " + listener.toString());
        listener.onNewBookArrived(book);
    }
    mListenerList.finishBroadcast();
}
```

在AndroidManifest中, Service与Activity不在同一进程.

``` xml
<!--与主应用不在同一进程中-->
<service
    android:name=".BookManagerService"
    android:process=":remote"/>
```

---

## 客户端

绑定服务和解绑服务, 绑定服务的具体内容, 都是在``mConnection``中实现.

``` java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);
    mTextView = (TextView) findViewById(R.id.main_tv_book_list);
}

@Override protected void onDestroy() {
    if (mRemoteBookManager != null && mRemoteBookManager.asBinder().isBinderAlive()) {
        try {
            Log.e(TAG, "解除注册");
            mRemoteBookManager.unregisterListener(mOnNewBookArrivedListener);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
    unbindService(mConnection);
    super.onDestroy();
}
```

添加内容, 注册监听接口.

``` java
private ServiceConnection mConnection = new ServiceConnection() {
    @Override public void onServiceConnected(ComponentName name, IBinder service) {
        IBookManager bookManager = IBookManager.Stub.asInterface(service);
        try {
            mRemoteBookManager = bookManager;
            Book newBook = new Book(3, "学姐的故事");
            bookManager.addBook(newBook);
            new BookListAsyncTask().execute();
            bookManager.registerListener(mOnNewBookArrivedListener);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override public void onServiceDisconnected(ComponentName name) {
        mRemoteBookManager = null;
        Log.e(TAG, "绑定结束");
    }
};
```

当调用监听接口时, 异步显示图书列表.

``` java
private IOnNewBookArrivedListener mOnNewBookArrivedListener = new IOnNewBookArrivedListener.Stub() {
    @Override public void onNewBookArrived(Book newBook) throws RemoteException {
        mHandler.obtainMessage(MESSAGE_NEW_BOOK_ARRIVED, newBook).sendToTarget();
    }
};

private Handler mHandler = new Handler() {
    @Override public void handleMessage(Message msg) {
        switch (msg.what) {
            case MESSAGE_NEW_BOOK_ARRIVED:
                Log.e(TAG, "收到的新书: " + msg.obj);
                new BookListAsyncTask().execute();
                break;
            default:
                super.handleMessage(msg);
                break;
        }
    }
};
```

点击``绑定服务``按钮, 执行绑定服务. 点击``获取图书数量``按钮, 获取当前列表的数量.

---

效果

![效果](https://raw.githubusercontent.com/SpikeKing/wcl-aidl-demo/master/articles/device-2016-06-12-140949.gif)

Android跨进程通信比较复杂, 但是意义重大, 目前常用的动态加载框架都需要处理跨进程通信等问题, 熟练基本原理, 掌握使用方式.

OK, that's all! Enjoy it!


