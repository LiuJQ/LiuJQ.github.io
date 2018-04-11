---
layout: post
title: Android 多进程 -- 浅析IPC
subtitle: On Android, one process cannot normally access the memory of another process.
tags: [Android, 多进程, Multi-Process]
---

&emsp;&emsp;在前一篇文章[Android 多进程 -- 揭开神秘面纱]({{ site.baseurl }}{% post_url 2018-03-30-android-multi-process %})中我们学习了如何在Android开发中使用多进程以及使用多进程需要注意的地方，那么伴随实现多进程而来的问题是，我们不得不解决跨进程之间的通信（IPC）问题。Android中解决IPC问题的方式多种多样，上一篇文章我们也总结过了，这次主要将介绍使用AIDL进行多进程通信，因为AIDL是Android提供给我们的标准跨进程通信API，非常灵活且强大。

&emsp;&emsp;跨进程通信的时候需要考虑一点，是否需要处理多线程并发的情况呢？

### Messenger
&emsp;&emsp;如果IPC不需要考虑多线程并发情况，那么Android提供了一种基于AIDL的轻量级IPC工具供开发者使用——Messenger。乍一看很熟悉，跟Handler处理的Message有关？对的，Messenger就是基于Handler传递Message的方式实现通信功能的，这也是为什么Messenger是线程安全（不支持并发）的原因。

&emsp;&emsp;查看源码不难发现Messenger是基于AIDL实现多进程通信的，Android SDK提供Messenger封装了Handler传递Message、然后通过IMessenger这个AIDL接口传递Message对象至Binder底层从而实现跨进程通信整个过程。

#### IMessenger AIDL定义
&emsp;&emsp;文件位置：frameworks/base/core/java/android/os/IMessenger.aidl
```java
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```

#### Messenger IPC流程
&emsp;&emsp;*注：下图来源于[Messenger 的工作原理](http://liwenkun.me/2017/02/25/how-does-messenger-work/)，侵删。*

![How does Messenger work ?](/assets/img/illustration/how_does_messenger_work.png)

&emsp;&emsp;图中清晰地展示了由客户端到服务端的单向跨进程通信流程，可以看到，其实Message对象说到底也是通过AIDL接口传递到Binder再发送到其他进程的。很多同学可能以为Messenger的跨进程通信是通过Handler实现，其实这个理解有点本末倒置了。Handler只能处理处于同一进程空间的Message消息，跨进程的Message是通过Binder传递的。Messenger实现IPC的过程中，Handler只是扮演了在服务端（或者客户端）的一个Message消息分发者和消费者角色。

#### Messenger如何使用
&emsp;&emsp;原理总是枯燥无味，代码更得人心。下面我们通过代码来演示如何使用Messenger实现跨进程通信（为便于演示，Client与Server处于同一个Application）。

&emsp;&emsp;[MessengerDemo源码](https://github.com/LiuJQ/MessengerDemo)
##### 服务端
&emsp;&emsp;先创建远程服务端DemoMessengerService，需要有一个Handler来传播和处理Message消息:
```java
private static class IncomingHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case DemoMessageConstants.MESSAGE_FROM_CLIENT:
                Log.d(TAG, "received message from client");
                break;
                default:
                super.handleMessage(msg);
        }
    }
}
```
&emsp;&emsp;重写Service的构造方法，初始化Messenger并传递Handler实例：
```java
public DemoMessengerService() {
    mMessenger = new Messenger(new IncomingHandler());
}
```
&emsp;&emsp;重写Service的onBind方法，返回Messenger的Binder实例：
```java
@Nullable
@Override
public IBinder onBind(Intent intent) {
    return mMessenger.getBinder();
}
```

##### 客户端
&emsp;&emsp;绑定远程服务Service：
```java
private final ServiceConnection mServiceConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        mMessenger = new Messenger(service);
        mBound = true;
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {
        mBound = false;
    }
};
// 绑定远程服务
bindService(new Intent(this, DemoMessengerService.class), mServiceConnection, BIND_AUTO_CREATE);
```
&emsp;&emsp;使用绑定后获取的Messenger实例发送消息：
```java
Message msg = Message.obtain(null, DemoMessageConstants.MESSAGE_FROM_CLIENT);
try {
    mMessenger.send(msg);
} catch (RemoteException e) {
    e.printStackTrace();
}
```

### AIDL
