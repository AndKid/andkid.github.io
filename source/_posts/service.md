---
title: Android Service
date: 2015-10-15 10:03:55
tags:
- android
- service
---
# Service

>Service是一类能够处理后台操作的应用组件，且能够用于支持跨进程调用。

## 启动模式
* Start 调用方-`startService()`，被调用方-`onStartCommand()`
* Bind  调用方-`bindService()`，被调用方-`IBind onBind()`

### Bind
绑定模式可以得到Service一个IBind接口，通过该接口实现对Service的远程调用，针对不同需求Service有三种定义该接口的方式。
* 继承Binder类  
  该方式用于自己的程序，将Service当作后台工作者，如果需要实现IPC，参考下面两种方式。  
  在Service中定义一个继承自Binder的类，在Client请求绑定时，Service通过onBinder将定义好的Binder对象返回给客户端供其调用Binder或Service的方法。  
* 利用Messenger  
  该方式可实现IPC，但只是以消息模式单线程地处理由其他进程发来的请求，若有多线程需求，参考最后一种方式。  
  在Service中定义Handler来处理Client的请求，该Handler基于可共享IBind给Client的Messenger对象，当Service和Client互相持有对方Messenger即可通过消息的方式通信。
* 利用AIDL  
  该方式允许不同应用跨进程访问Service，并且支持多线程来异步提供服务。  
  除了android提供的系统服务，一般应用没必要使用这种方式，利用Messenger更为便捷。

##### 利用Messenger
Service端
```java
public class MessengerService extends Service {
    /** Command to the service to display a message */
    static final int MSG_SAY_HELLO = 1;

    /**
     * Handler of incoming messages from clients.
     */
    class IncomingHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SAY_HELLO:
                    Toast.makeText(getApplicationContext(), "hello!", Toast.LENGTH_SHORT).show();
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    /**
     * Target we publish for clients to send messages to IncomingHandler.
     */
    final Messenger mMessenger = new Messenger(new IncomingHandler());

    /**
     * When binding to the service, we return an interface to our messenger
     * for sending messages to the service.
     */
    @Override
    public IBinder onBind(Intent intent) {
        Toast.makeText(getApplicationContext(), "binding", Toast.LENGTH_SHORT).show();
        return mMessenger.getBinder();
    }
}
```
Client端
```java
public class ActivityMessenger extends Activity {
    /** Messenger for communicating with the service. */
    Messenger mService = null;

    /** Flag indicating whether we have called bind on the service. */
    boolean mBound;

    /**
     * Class for interacting with the main interface of the service.
     */
    private ServiceConnection mConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
            // This is called when the connection with the service has been
            // established, giving us the object we can use to
            // interact with the service.  We are communicating with the
            // service using a Messenger, so here we get a client-side
            // representation of that from the raw IBinder object.
            mService = new Messenger(service);
            mBound = true;
        }

        public void onServiceDisconnected(ComponentName className) {
            // This is called when the connection with the service has been
            // unexpectedly disconnected -- that is, its process crashed.
            mService = null;
            mBound = false;
        }
    };

    public void sayHello(View v) {
        if (!mBound) return;
        // Create and send a message to the service, using a supported 'what' value
        Message msg = Message.obtain(null, MessengerService.MSG_SAY_HELLO, 0, 0);
        try {
            mService.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        // Bind to the service
        bindService(new Intent(this, MessengerService.class), mConnection,
            Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        // Unbind from the service
        if (mBound) {
            unbindService(mConnection);
            mBound = false;
        }
    }
}
```
>注：Messenger本身也是通过AIDL实现  
疑问，AIDL的多线程是如何实现的，Messenger是通过AIDL的方式实现，为什么不支持多线程？

##### 利用AIDL
AIDL的方式只适用于IPC，本地调用请使用继承Binder的方式。由于AIDL支持多线程，需要考虑到线程安全问题。  
AIDL可设置`oneway`工作模式，该模式在Client调用后可立刻返回，异步调用。

###### 定义AIDL步骤
1. 创建.aidl文件
```java
    // IRemoteService.aidl
    package com.example.android;

    // Declare any non-default types here with import statements

    /** Example service interface */
    interface IRemoteService {
        /** Request the process ID of this service, to do evil things with it. */
        int getPid();

        /** Demonstrates some basic types that you can use as parameters
         * and return values in AIDL.
         */
        void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
                double aDouble, String aString);
    }
```
2. 实现接口方法
```java
    private final IRemoteService.Stub mBinder = new IRemoteService.Stub() {
        public int getPid(){
            return Process.myPid();
        }
        public void basicTypes(int anInt, long aLong, boolean aBoolean,
            float aFloat, double aDouble, String aString) {
            // Does nothing
        }
    };
```
3. 把接口通过onBind暴露给Client
```java
    public class RemoteService extends Service {
        @Override
        public void onCreate() {
            super.onCreate();
        }

        @Override
        public IBinder onBind(Intent intent) {
            // Return the interface
            return mBinder;
        }

        private final IRemoteService.Stub mBinder = new IRemoteService.Stub() {
            public int getPid(){
                return Process.myPid();
            }
            public void basicTypes(int anInt, long aLong, boolean aBoolean,
                float aFloat, double aDouble, String aString) {
                // Does nothing
            }
        };
    }
```
4. Client引用
```java
    IRemoteService mIRemoteService;
    private ServiceConnection mConnection = new ServiceConnection() {
        // Called when the connection with the service is established
        public void onServiceConnected(ComponentName className, IBinder service) {
            // Following the example above for an AIDL interface,
            // this gets an instance of the IRemoteInterface, which we can use to call on the service
            mIRemoteService = IRemoteService.Stub.asInterface(service);
        }

        // Called when the connection with the service disconnects unexpectedly
        public void onServiceDisconnected(ComponentName className) {
            Log.e(TAG, "Service has unexpectedly disconnected");
            mIRemoteService = null;
        }
    };
```

###### 传递对象
AIDL默认支持基本数据类型参数，如果要传递复杂类型对象，该对象需要实现Parcelable接口并重写相应方法
```java
    import android.os.Parcel;
    import android.os.Parcelable;

    public final class Rect implements Parcelable {
        public int left;
        public int top;
        public int right;
        public int bottom;

        public static final Parcelable.Creator<Rect> CREATOR = new
    Parcelable.Creator<Rect>() {
            public Rect createFromParcel(Parcel in) {
                return new Rect(in);
            }

            public Rect[] newArray(int size) {
                return new Rect[size];
            }
        };

        public Rect() {
        }

        private Rect(Parcel in) {
            readFromParcel(in);
        }

        public void writeToParcel(Parcel out) {
            out.writeInt(left);
            out.writeInt(top);
            out.writeInt(right);
            out.writeInt(bottom);
        }

        public void readFromParcel(Parcel in) {
            left = in.readInt();
            top = in.readInt();
            right = in.readInt();
            bottom = in.readInt();
        }
    }
```
创建与该类型对应的.aidl文件
```java
    package android.graphics;

    // Declare Rect so AIDL can find it and knows that it implements
    // the parcelable protocol.
    parcelable Rect;
```
最后在需要引用该类型的aidl文件中也要import该包  
