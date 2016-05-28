---
title: Android Messager
date: 2015-10-18 14:11:32
tags:
- android
- messager
- IPC
---
# Messager

>Reference to a Handler, which others can use to send messages to it. This allows for the implementation of message-based communication across processes, by creating a Messenger pointing to a Handler in one process, and handing that Messenger to another process.（Messager对象指向一个Handler，其他进程可利用Messager发消息给Handler。
实现了基于消息传递的进程间通信，通过在一个进程中创建指向Handler的Messager，并把这个Messager暴露给另一个进程。）

```java
private final Messenger mMessenger = new Messenger(mIncomingHandler);//创建指向名为mIncomingHandler的Handler的Messager

当前进程与服务之间的ServiceConnection对象
private ServiceConnection mConnection = new ServiceConnection() {
	public void onServiceConnected(ComponentName name, IBinder service) {
	    mHealthServiceBound = true;
	    Message msg = Message.obtain(null, BluetoothHDPService.MSG_REG_CLIENT);
	    msg.replyTo = mMessenger;//将Messager添加到附给msg.replyTo
	    mHealthService = new Messenger(service);//利用IBinder创建服务的Messager
	    try {
		mHealthService.send(msg);//利用服务的Messager发送消息给服务，服务端得到msg.replayTo的代表当前进程的Messager即可发送消息给当前进程指定的Handler
	    } catch (RemoteException e) {
		Log.w(TAG, "Unable to register client to service.");
		e.printStackTrace();
	    }
	}

	public void onServiceDisconnected(ComponentName name) {
	    mHealthService = null;
	    mHealthServiceBound = false;
	}
};

在当前进程启动并绑定服务
Intent intent = new Intent(this, BluetoothHDPService.class);
startService(intent);
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```

(后来发现以上所写Messager实现的未必是进程间通信，因为Service在默认情况下在UI线程内的)  
(应该可以在android manifest中将该service的process属性设成别的或者直接调用其他应用的service)
