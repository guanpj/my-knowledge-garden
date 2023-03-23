
AIDL 是 Android Interface Definition Language（Android 接口定义语言）的缩写，它是 Android 进程间通信的接口语言。由于 Android 系统的 Linux 内核采用了进程隔离机制，使得不同的应用程序运行在不同的进程当中，有时候两个应用之间需要传递或者共享某些数据，就需要进行进程间的通信讯。

在上一篇文章——[《Binder 机制分析——原理篇》](https://ywue4d2ujm.feishu.cn/docs/doccnnNIKMdICh9atjwTbBRYxpf)中我们已经分析了使用 Binder 机制的原因以及分析了 Binder 机制，而 AIDL 也正是运用了 Binder 机制来实现进程间的通讯，本章我们将继续从 AIDL 的使用过程体验 Binder 在应用层的使用和原理。

# AIDL 使用步骤

## 1、创建 .aidl 接口文件

首先在 aidl 文件夹这种新建文件并且命名为 IMyService.aidl<em>，</em>并且按照以下格式添加方法。此举是为了声明作为 Server 端的远程 Service 具有哪些能力：

```java
package com.me.guanpj.binder;

import com.me.guanpj.binder.User;
// Declare any non-default types here with import statements

interface IMyService {
    void addUser(in User user);

    List<User> getUserList();
}
```

对于对象引用，还需要引入实体类 User.aidl<em>：</em>

```java
// User.aidl
package com.me.guanpj.binder;

// Declare any non-default types here with import statements

parcelable User;
```

跨进程传输的 User 实体必须实现 Parcelable 接口：

```java
public class User implements Parcelable {
    public int id;
    public String name;

    public User() {}

    public User(int id, String name) {
        this.id = id;
        this.name = name;
    }

    protected User(Parcel in) {
        id = in.readInt();
        name = in.readString();
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
    }

    @Override
    public int describeContents() {
        return 0;
    }

    public static final Creator<User> CREATOR = new Creator<User>() {
        @Override
        public User createFromParcel(Parcel in) {
            return new User(in);
        }

        @Override
        public User[] newArray(int size) {
            return new User[size];
        }
    };
}
```

在创建 IMyService.aidl 文件并且 ReBuild 项目之后，IDE 会自动在 build 目录生成一个与 IMyService.java 接口类：

```java
package com.me.guanpj.binder;
// Declare any non-default types here with import statements

public interface IMyService extends android.os.IInterface
{
    /** Local-side IPC implementation stub class. */
    public static abstract class Stub extends android.os.Binder implements com.me.guanpj.binder.IMyService
    {
        private static final java.lang.String DESCRIPTOR = "com.me.guanpj.binder.IMyService";
        /** Construct the stub at attach it to the interface. */
        public Stub()
        {
            this.attachInterface(this, DESCRIPTOR);
        }
        /**
         * Cast an IBinder object into an com.me.guanpj.binder.IMyService interface,
         * generating a proxy if needed.
         */
        public static com.me.guanpj.binder.IMyService asInterface(android.os.IBinder obj)
        {
            if ((obj==null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin!=null)&&(iin instanceof com.me.guanpj.binder.IMyService))) {
                return ((com.me.guanpj.binder.IMyService)iin);
            }
            return new com.me.guanpj.binder.IMyService.Stub.Proxy(obj);
        }
        @Override public android.os.IBinder asBinder()
        {
            return this;
        }
        @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
        {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code)
            {
                case INTERFACE_TRANSACTION:
                {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_addUser:
                {
                    data.enforceInterface(descriptor);
                    com.me.guanpj.binder.User _arg0;
                    if ((0!=data.readInt())) {
                        _arg0 = com.me.guanpj.binder.User.CREATOR.createFromParcel(data);
                    }
                    else {
                        _arg0 = null;
                    }
                    this.addUser(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_getUserList:
                {
                    data.enforceInterface(descriptor);
                    java.util.List<com.me.guanpj.binder.User> _result = this.getUserList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                default:
                {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }
        private static class Proxy implements com.me.guanpj.binder.IMyService
        {
            private android.os.IBinder mRemote;
            Proxy(android.os.IBinder remote)
            {
                mRemote = remote;
            }
            @Override public android.os.IBinder asBinder()
            {
                return mRemote;
            }
            public java.lang.String getInterfaceDescriptor()
            {
                return DESCRIPTOR;
            }
            @Override public void addUser(com.me.guanpj.binder.User user) throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((user!=null)) {
                        _data.writeInt(1);
                        user.writeToParcel(_data, 0);
                    }
                    else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addUser, _data, _reply, 0);
                    _reply.readException();
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            @Override public java.util.List<com.me.guanpj.binder.User> getUserList() throws android.os.RemoteException
            {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.me.guanpj.binder.User> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getUserList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.me.guanpj.binder.User.CREATOR);
                }
                finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }
        static final int TRANSACTION_addUser = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_getUserList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }
    public void addUser(com.me.guanpj.binder.User user) throws android.os.RemoteException;
    public java.util.List<com.me.guanpj.binder.User> getUserList() throws android.os.RemoteException;
}
```

它继承了 IInterface 接口，IMyService 接口只有一个静态抽象类 Stub，它继承自 Binder 并实现了 IMyService 接口。

Stub 里面也有一个静态内部类 Proxy，它也继承了 IMyService（是不是有点乱，乱就对了，我也很乱）。如此嵌套是为了避免有多个 .aidl 文件的时候自动生成这些类的类名不会重复。

为了提高代码可读性，这里将生成的 Stub 内部类单独抽取到文件 Stub.java 文件中，并在关键方法上添加了注释和 Log：

```java
public abstract class Stub extends Binder implements IMyService {
    /**
     * Construct the mLocalStub at attach it to the interface.
     */
    public Stub() {
        this.attachInterface(this, DESCRIPTOR);
    }

    /**
     * 根据 Binder 本地对象或者代理对象返回 IMyServcie 接口
     */
    public static IMyService asInterface(android.os.IBinder obj) {
        if ((obj == null)) {
            return null;
        }
        //查找本地对象
        android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
        if (((iin != null) && (iin instanceof IMyService))) {
            Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————返回本地对象");
            return ((IMyService) iin);
        }
        Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————返回代理对象");
        return new Stub.Proxy(obj);
    }

    @Override
    public android.os.IBinder asBinder() {
        return this;
    }

    @Override
    public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
        switch (code) {
            case INTERFACE_TRANSACTION: {
                reply.writeString(DESCRIPTOR);
                return true;
            }
            case TRANSACTION_addUser: {
                Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————本地对象通过 Binder 执行 addUser");
                data.enforceInterface(DESCRIPTOR);
                User arg0;
                if ((0 != data.readInt())) {
                    //取出客户端传递过来的数据
                    arg0 = User.CREATOR.createFromParcel(data);
                } else {
                    arg0 = null;
                }
                //调用 Binder 本地对象
                this.addUser(arg0);
                reply.writeNoException();
                return true;
            }
            case TRANSACTION_getUserList: {
                Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————本地对象通过 Binder 执行 getUserList");
                data.enforceInterface(DESCRIPTOR);
                //调用 Binder 本地对象
                List<User> result = this.getUserList();
                reply.writeNoException();
                //将结果返回给客户端
                reply.writeTypedList(result);
                return true;
            }
            default:
                break;
        }
        return super.onTransact(code, data, reply, flags);
    }


    private static class Proxy implements IMyService {
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

        @Override
        public void addUser(User user) throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
               if (user != null) {
                   _data.writeInt(1);
                   //将 user 对象的值写入 _data
                   user.writeToParcel(_data, 0);
               } else {
                   _data.writeInt(0);
               }
               Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————代理对象通过 Binder 调用 addUser");
               //通过 transact 跟 Server 交互
               mRemote.transact(Stub.TRANSACTION_addUser, _data, _reply, 0);
                _reply.readException();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
        }

        @Override
        public List<User> getUserList() throws android.os.RemoteException {
            android.os.Parcel _data = android.os.Parcel.obtain();
            android.os.Parcel _reply = android.os.Parcel.obtain();
            List<User> _result;
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————代理对象通过 Binder 调用 getUserList");
                //通过 transact 跟 Server 交互
                mRemote.transact(Stub.TRANSACTION_getUserList, _data, _reply, 0);
                _reply.readException();
                //获取 Server 的返回值并进程转换
                _result = _reply.createTypedArrayList(User.CREATOR);
            } finally {
                _reply.recycle();
                _data.recycle();
            }
            return _result;
        }
    }
}
```

## 2、创建 Service

创建 UserServer 类并继承自 Service，并且创建一个内部类 IMyServiceNative 并且继承 Stub 类，然后将该实现类的实例在 onBind 方法返回：

```java
public class UserServer extends Service {

    class IMyServiceNative extends Stub {

        List<User> users = new ArrayList<>();

        @Override
        public void addUser(User user) {
            Log.e("gpj", "进程：" + Utils.getProcessName(getApplicationContext())
                    + "，线程：" + Thread.currentThread().getName() + "————Server 执行 addUser");
            users.add(user);
        }

        @Override
        public List<User> getUserList() {
            Log.e("gpj", "进程：" + Utils.getProcessName(getApplicationContext())
                    + "，线程：" + Thread.currentThread().getName() + "————Server 执行 getUserList");
            return users;
        }
    }

    private IMyServiceNative mMyServcieNative = new IMyServiceNative();

    @Override
    public IBinder onBind(Intent intent) {
        Log.e("gpj", "进程：" + Utils.getProcessName(getApplicationContext())
                + "，线程：" + Thread.currentThread().getName() + "————Server onBind");
        return mMyServcieNative;
    }
}
```

因为 Stub 抽象类实现了 IMyService 接口，它声明了 Server 端具有的能力。因此在 IMyServiceNative 中应该实现 IMyService 接口中声明的 addUser 和 getUserList 方法。

除此之外，还需要在 Menifest 文件中声明 MyService 并标记为远程 Service：

```xml
<service android:name=".MyService"
    android:process=":remote">
    <intent-filter>
        <action android:name="com.me.guanpj.binder"/>
    </intent-filter>
</service>
```

## 3、绑定 Service

作为客户端，在 ClientActivity 中绑定远程 Service：

```java
private void bindService() {
    Intent intent = new Intent();
    intent.setAction("com.me.guanpj.binder");
    intent.setComponent(new ComponentName("com.me.guanpj.binder", "com.me.guanpj.binder.MyService"));

    Log.e("gpj", "进程：" + Utils.getProcessName(getApplicationContext())
            + "，线程：" + Thread.currentThread().getName() + "————开始绑定 Servcie");
    bindService(intent, mConn, Context.BIND_AUTO_CREATE);
}

IMyService myService;

private ServiceConnection mConn = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        Log.e("gpj", "进程：" + Utils.getProcessName(getApplicationContext())
                + "，线程：" + Thread.currentThread().getName() + "————Client onServiceConnected");
        myService = Stub.asInterface(service);
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {
        myService = null;
    }
};
```

在 onServiceConnected 回调方法中，将 Binder 驱动返回的 service 包装成 Stub 对象并赋值给 myService，随后就可以使用 myService 实例与 Server 端交互了。

## 4、Client 端与 Server 端交互

绑定远程 Service 并且获取到 Server 端的代理对象后，Client 端便可通过代理对象与 Server 端进行通讯了：

```java
@Override
public void onClick(View v) {
    switch (v.getId()) {
        case R.id.btn_bind:
            bindService();
            break;
        case R.id.btn_add_user:
            if (null != myService) {
                try {
                    Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————Client 调用 addUser");
                    myService.addUser(new User(111, "gpj"));
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            } else {
                Toast.makeText(ClientActivity.this, "先绑定 Service 才能调用方法", Toast.LENGTH_LONG).show();
            }
            break;
        case R.id.btn_get_size:
            if (null != myService) {
                try {
                    Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————Client 调用 getUserList");
                    List<User> userList = myService.getUserList();
                    tvResult.setText("getUserList size:" + userList.size());
                    Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————UserList Size：" + userList.size());
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            } else {
                Toast.makeText(ClientActivity.this, "先绑定 Service 才能调用方法", Toast.LENGTH_LONG).show();
            }
            break;
        default:
    }
}
```

## 5、结果展示

经过以上步骤，处于 Client 端的 Activity 与作为服务端的远程 Service 虽然运行在不同的进程中，通过 AIDL 生成的模板代码并且依赖 Binder 机制，它们之间实现了数据的传递。

运行结果如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044626.png)

首先点击 BindService 按钮，此时会绑定远程 Service 并且获取到服务端的代理对象 myService：

```plain text
E/gpj: 进程：com.me.guanpj.binder，线程：main————开始绑定 Servcie
E/gpj: 进程：com.me.guanpj.binder:remote，线程：main————Server onBind
E/gpj: 进程：com.me.guanpj.binder，线程：main————Client onServiceConnected
E/gpj: 进程：com.me.guanpj.binder，线程：main————返回代理对象
```

点击 Add User 按钮往 Server 端增加一名用户：

```plain text
E/gpj: 进程：com.me.guanpj.binder，线程：main————Client 调用 addUser
E/gpj: 进程：com.me.guanpj.binder，线程：main————代理对象通过 Binder 调用 addUser
E/gpj: 进程：com.me.guanpj.binder:remote，线程：Binder:4072_3————本地对象通过 Binder 执行 addUser
E/gpj: 进程：com.me.guanpj.binder:remote，线程：Binder:4072_3————Server 执行 addUser:User{id=111, name='gpj'}
```

点击 Get Size 按钮查询用户数量：

```plain text
E/gpj: 进程：com.me.guanpj.binder，线程：main————Client 调用 getUserList
E/gpj: 进程：com.me.guanpj.binder，线程：main————代理对象通过 Binder 调用 getUserList
E/gpj: 进程：com.me.guanpj.binder:remote，线程：Binder:4072_3————本地对象通过 Binder 执行 getUserList
E/gpj: 进程：com.me.guanpj.binder:remote，线程：Binder:4072_3————Server 执行 getUserList
E/gpj: 进程：com.me.guanpj.binder，线程：main————UserList Size：1
```

# AIDL 通讯过程分析

## 关键类职责描述

以上代码已经上传到了 [Github](https://github.com/guanpj/BinderDemo)，关键的类以及它们之间的关系如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044631.png)

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044634.png)

- <strong>IInterface:</strong> 声明（自动生成或者手动创建）AIDL 性质的接口必须继承这个接口，这个接口只有一个 IBinder asBinder() 方法，实现它的类代表它能够进程跨进程传输（ Server 端的 Binder 本地对象）或者持有能够进程跨进程传输的对象的引用（Binder 代理对象）。
- <strong>I</strong><strong>MyServcie</strong><strong>:</strong> 如 1 中所述，它需要继承 IInterface 接口，并且定义方法（声明 Server 端具有的能力： addUser 和 getUserList）。
- <strong>IBinder:</strong> 它也是一个接口，实现这个接口的对象就具有了跨进程传输的能力（由驱动底层支持），在跨进程数据流经驱动的时候，驱动会识别 IBinder 类型的数据，从而自动完成不同进程 Binder 本地对象以及 Binder 代理对象的转换。
- <strong>Binder:</strong> Java 层的 Binder 类，代表 Binder 本地对象。
- <strong>BinderProxy</strong><strong>: </strong>Binder 类的内部类，是 Server 端 Binder 对象的本地代理，它们都继承了 IBinder 接口，因此都能跨进程进行传输，Binder 驱动在跨进程传输的时候会将这两个对象自动进行转换。
- <strong>Stub</strong><strong>:</strong> 由 IDE 自动生成的抽象类，字面意义为“存根”，这里指待服务端实现的意思。它继承自 Binder 并实现了 IInterface 接口，说明它是 Server 端的 Binder 本地对象，并拥有 Server 承诺给 Client 的能力。
- <strong>ClientActivity:</strong> 在这里充当 Client 端的门面，它持有 IMyService 接口的实例 myService，并且通过 myService 与服务端进行通讯。
- <strong>UserServer:</strong> 它是一个 Service 组件，可以运行在独立的进程中。当作为远程 Service 需要与 Client 端进行绑定时，必须在 onBind 方法中返回一个 IBinder 对象，Client 端才能与之进行交互。
- <strong>MyServiceNative: </strong>Server 端的 Binder 本地对象。它实现了 IMyServcie 声明的能力（方法），Client 可以间接调用到它从而实现通讯功能

## 通讯过程分析

一次跨进程通信必然会涉及到两个进程，在这个例子中 UserServer 处于服务端进程，使用内部类 MyServiceNative<strong> </strong>提供服务；ClientActivity 作为客户端进程，使用 UserServer 提供的服务。

先从 ClientActivity 中绑定服务后的回调方法着手：

```java
public class ClientActivity extends AppCompatActivity {
    ...
    private ServiceConnection mConn = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.e("gpj", "进程：" + Utils.getProcessName()
                    + "，线程：" + Thread.currentThread().getName() + "————Client onServiceConnected");
            myService = Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            myService = null;
        }
    };
    ...
}
```

onServiceConnected 的参数中，第一个是 Service 组件的名字，表示哪个服务被启动了；

重点是类型为 IBinder 的第二个参数，在 UserServer 中的 onBind 方法中，已经把 Server 端的本地对象 MyServiceNative  的实例返回给 Binder 驱动了：

```java
public class UserServer extends Service {
    ...
    private MyServiceNative mMyServcieNative = new MyServiceNative();

    @Override
    public IBinder onBind(Intent intent) {
        Log.e("gpj", "进程：" + Utils.getProcessName()
                + "，线程：" + Thread.currentThread().getName() + "————Server onBind");
        return mMyServcieNative;
    }
}
```

因此，当该服务被绑定的时候，Binder 驱动会为根据该服务所在的进程决定是返回本地对象还是代理对象给客户端：

当 Service 与 ClientActivity 位于同一个进程当中的时候，onServiceConnected 返回 Binder 本地对象——即 MyServcieNative 实例给客户端：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044640.png)

当 Service 运行在不同进程中的时候（Manifest 中声明 Service 的时候设置 proces 属性），返回的是 BinderProxy 实例：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044643.png)

接着会将这个 IBinder 实例传给 Stub 的 asInterface 方法：

```java
/**
 * 根据 Binder 本地对象或者代理对象返回 IMyService 接口
 */
public static IMyService asInterface(android.os.IBinder obj) {
    if ((obj == null)) {
        return null;
    }
    //查找本地对象
    android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
    if (((iin != null) && (iin instanceof IMyService))) {
        Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————返回本地对象");
        return ((IMyService) iin);
    }
    Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————返回代理对象");
    return new Stub.Proxy(obj);
}
```

首先，会根据 DESCRIPTOR 调用 IBinder 对象的 queryLocalInterface 方法，那么就得看 IBinder 的实现类怎么处理这个方法了：

<strong>在 Binder 类中的实现：</strong>

```java
public @Nullable IInterface queryLocalInterface(@NonNull String descriptor) {
    //判断 mDescriptor 跟参数 DESCRIPTOR 相同，返回 mOwner
    if (mDescriptor != null && mDescriptor.equals(descriptor)) {
        return mOwner;
    }
    return null;
}
```

那么这个 mOwner 和 mDescriptor 又是什么时候被赋值的呢？答案在 Binder 的子类 Stub 的构造方法里面，：

```java
public Stub() {
    //将 Stub 和 DESCRIPTOR 注入到父类（Binder）
    this.attachInterface(this, DESCRIPTOR);
}
```

<strong>在 Binder$BinderProxy 类中的实现：</strong>

BinderProxy 并不是 Binder 本地对象，而是 Binder 的本地代理，因此 queryLocalInterface 返回的是 null：

```java
public IInterface queryLocalInterface(String descriptor) {
    return null;
}
```

根据以上分析可以得出结论：

当 UserServer 与客户端处于相同进程时，obj.queryLocalInterface(DESCRIPTOR) 方法存在返回值并且是 IMyService 实例，而它就是 MyServiceNative，将它直接返回给 Client 调用；

处于不同进程时，obj.queryLocalInterface(DESCRIPTOR) 返回 null，这时使用 Stub$Proxy 类将其进行包装后再返回。Proxy 类也实现了 IMyService 接口，因此，在 Client 看来，它也具有 Server 承诺给 Client 的能力。

那么，经过包装后的 Stub$Proxy 对象怎么和 Server 进行交互呢？首先，它会把 BinderProxy 对象保存下来：

```java
private android.os.IBinder mRemote;

Proxy(android.os.IBinder remote) {
    mRemote = remote;
}
```

然后，实现 IMyService 的方法：

```java
@Override
public void addUser(User user) throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        if (user != null) {
            _data.writeInt(1);
            //将 user 对象的值写入 _data
            user.writeToParcel(_data, 0);
        } else {
            _data.writeInt(0);
        }
        Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————代理对象通过 Binder 调用 addUser");
        //通过 transact 跟 Server 交互
        mRemote.transact(Stub.TRANSACTION_addUser, _data, _reply, 0);
        _reply.readException();
    } finally {
        _reply.recycle();
        _data.recycle();
    }
}

@Override
public List<User> getUserList() throws android.os.RemoteException {
    android.os.Parcel _data = android.os.Parcel.obtain();
    android.os.Parcel _reply = android.os.Parcel.obtain();
    List<User> _result;
    try {
        _data.writeInterfaceToken(DESCRIPTOR);
        Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————代理对象通过 Binder 调用 getUserList");
        //通过 transact 跟 Server 交互
        mRemote.transact(Stub.TRANSACTION_getUserList, _data, _reply, 0);
        _reply.readException();
        //获取 Server 的返回值并进程转换
        _result = _reply.createTypedArrayList(User.CREATOR);
    } finally {
        _reply.recycle();
        _data.recycle();
    }
    return _result;
}
```

可以看到，不管什么方法，都是是将服务端的方法代号、处理过的参数和接收返回值的对象等通过 mRemote.transact() 方法 Server 进行交互，mRemote 是 BinderProxy 类型，在 BinderProxy 类中，最终调用的是 transactNative 方法：

```java
public native boolean transactNative(int code, Parcel data, Parcel reply, int flags) throws RemoteException;
```

它的最终实现在 Native 层进行，Binder 驱动会通过 ioctl 系统调用唤醒 Server 进程，并调用本地对象 MyServiceNative 的 onTransact 函数（在 Stub 类中）：

```java
@Override
public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
    switch (code) {
        case INTERFACE_TRANSACTION: {
            reply.writeString(DESCRIPTOR);
            return true;
        }
        case TRANSACTION_addUser: {
            Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————本地对象通过 Binder 执行 addUser");
            data.enforceInterface(DESCRIPTOR);
            User arg0;
            if ((0 != data.readInt())) {
                //取出客户端传递过来的数据
                arg0 = User.CREATOR.createFromParcel(data);
            } else {
                arg0 = null;
            }
            //调用 Binder 本地对象
            this.addUser(arg0);
            reply.writeNoException();
            return true;
        }
        case TRANSACTION_getUserList: {
            Log.e("gpj", "线程：" + Thread.currentThread().getName() + "————本地对象通过 Binder 执行 getUserList");
            data.enforceInterface(DESCRIPTOR);
            //调用 Binder 本地对象
            List<User> result = this.getUserList();
            reply.writeNoException();
            //将结果返回给客户端
            reply.writeTypedList(result);
            return true;
        }
        default:
            break;
    }
    return super.onTransact(code, data, reply, flags);
}
```

在 Server 进程中，onTransact 会根据 Client 传过来的方法代号决定调用哪个方法，得到结果后又会通过 Binder 驱动返回给 Client。

回溯到 onServiceConnected 回调方法，待服务连接成功后，Client 就需要跟 Server 进行交互了，如果 Server 跟 Client 在同一个进程中，Client 可以直接调用 Server 的本地对象 ，当它们不在同一个进程中的时候，Binder 驱动会自动将 Server 的本地对象转换成 BinderProxy 代理对象，经过一层包装之后，返回一个新的代理对象给 Client。这样，整个 IPC 的过程就完成了。

整个过程的流程如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044649.png)

# Binder 在 Framework 层的应用

在 Activity 启动的初始阶段的流程如下：

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044652.png)

其中，最后一段中的 ActivityManagerProxy 与 ActivityManagerServer 分别处于不同的进程中，它们之间的通讯也依赖 Binder 机制。

这里从 Instrumentation 类的 execStartActivity() 方法开始分析：

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
        ...
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        // 获取 AMS 的代理对象并调用其 startActivity 方法
        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

以上过程是在 Launcher App 所在的进程中发生的，AMS 是运行在 system_server 进程中的，这时 AMS 就相当于 AIDL 中的远程 Service，App 进程要与 AMS 交互，需要通过 AMS 的代理对象 AMP(ActivityManagerProxy) 来完成，来看 ActivityManagerNative.getDefault() 拿到的是什么：

```java
static public IActivityManager getDefault() {
    return gDefault.get();
}
```

getDefault 是一个静态变量：

```java
private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
    protected IActivityManager create() {
        // 向 ServiceManager 查询一个 key 为 "activity" 的引用
        IBinder b = ServiceManager.getService("activity");
        if (false) {
            Log.v("ActivityManager", "default service binder = " + b);
        }
        IActivityManager am = asInterface(b);
        if (false) {
            Log.v("ActivityManager", "default service = " + am);
        }
        return am;
    }
};
```

这里通过 "activity" 这个名字向 ServiceManager 查询 AMS 的引用（事实上这个过程也是通过 Binder 来实现的，这里不再分析），获取 AMS 的引用后，调用 asInterface 方法：

```java
static public IActivityManager asInterface(IBinder obj) {
    if (obj == null) {
        return null;
    }
    // 根据 descriptor 查询 obj 是否为 Binder 本地对象，具体过程请看前文中提到的文章
    IActivityManager in = (IActivityManager)obj.queryLocalInterface(descriptor);
    if (in != null) {
        return in;
    }
    // 如果 obj 不是 Binder 本地对象，则将其包装成代理对象并返回
    return new ActivityManagerProxy(obj);
}
```

因为 AMS 与 Launcher App 不在同一个进程中，这里返回的 IBinder 对象是一个 Binder 代理对象，因此这类将其包装成 AMP(ActivityManagerProxy) 对象并返回，AMP 是 AMN(ActivityManagerNative) 的内部类，查看 AMP 类 ：

```java
class ActivityManagerProxy implements IActivityManager
{
    public ActivityManagerProxy(IBinder remote)
    {
        mRemote = remote;
    }

    public IBinder asBinder()
    {
        return mRemote;
    }

    public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        ...
        // 调用号为 START_ACTIVITY_TRANSACTION
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
    ...
    public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, String callingPackage, int userId) throws RemoteException
    {
        ...
        mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();
        ComponentName res = ComponentName.readFromParcel(reply);
        data.recycle();
        reply.recycle();
        return res;
    }
    ...
}
```

可以看到，AMP 里面将客户端的请求通过 mRemote.transact() 进行转发；mRemote 正是 Binder 驱动返回来的 BinderProxy 对象的实例，通过 mRemote，Binder 驱动最终将调用处于 Binder Server 端 AMN 中的 onTransact 方法：

```java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    // 根据方法调用号 code 决定调用哪个方法
    switch (code) {
    case START_ACTIVITY_TRANSACTION:
    {
        ...
        // 调用 startActivity 方法
        int result = startActivity(app, callingPackage, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, options);
        reply.writeNoException();
        reply.writeInt(result);
        return true;
    }
    ...
    case START_SERVICE_TRANSACTION: {
        ...
        ComponentName cn = startService(app, service, resolvedType, callingPackage, userId);
            reply.writeNoException();
            ComponentName.writeToParcel(cn, reply);
            return true;
        }
        ...
    }
}
```

AMN 是一个抽象类，它的 startActivity 为抽象方法，具体的实现在 ActivityManagerService.java 中:

```java
public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
    ...
    
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
                                   Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
                                   int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
    
    ...
}
```

这样最终从 App 进程调用到了 AMS 中的方法，剩下的流程这里不再进行分析。

# 总结

![](https://my-bucket-1251125515.cos.ap-guangzhou.myqcloud.com/Binder-2/clipboard_20230323_044658.png)

<strong>参考文章</strong>

[写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)

[Binder 学习指南](https://cloud.tencent.com/developer/article/1329601)

> 文章中的代码已经上传至我的 Github，如果你对文章内容有疑问或者有不同的意见，欢迎留言，我们一同探讨。
