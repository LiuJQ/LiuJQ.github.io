---
layout: post
title: Android 多进程 -- 浅析IPC
subtitle: On Android, one process cannot normally access the memory of another process.
feature-img: "assets/img/pexels/old-phone.jpeg"
tags: [Android, 多进程, Multi-Process]
---

&emsp;&emsp;在前一篇文章 [Android 多进程 -- 揭开神秘面纱]({{ site.baseurl }}{% post_url 2018-03-30-android-multi-process %}) 中我们学习了如何在Android开发中使用多进程以及使用多进程需要注意的地方，那么伴随实现多进程而来的问题是，我们不得不解决跨进程之间的通信（IPC）问题。Android中解决IPC问题的方式多种多样，上一篇文章我们也总结过了，这次主要将介绍使用AIDL进行多进程通信，因为AIDL是Android提供给我们的标准跨进程通信API，非常灵活且强大。

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

&emsp;&emsp;AIDL(Android Interface Definition Language)，顾名思义，它只是一种Android接口定义语言，我们无法直接把它当作IPC的工具来使用，但是我们可以通过AIDL定义IPC所需要的接口，然后编译器会帮我们自动生成IPC的一套模板源码。从编程语言的角度出发，我们也可以把AIDL当作是一种模板接口，利用它可以编写出IPC所需要的接口而又省略大部分相似的代码。

**AIDL支持的数据类型**
* Java 编程语言中的所有基本数据类型（如 int、long、char、boolean 等等）
* String
* CharSequence
* Parcelable：实现了Parcelable接口的对象
* List：元素须是AIDL支持的数据类型或其他 AIDL 接口；接口定义使用 List 通用类型，另一端实际接收的具体类是 ArrayList
* Map：key 和 value 须是AIDL支持的数据类型或其他 AIDL 接口；接口定义时使用 Map 通用类型，另一端实际接收的具体类是 HashMap

**AIDL其他注意事项**
* 在AIDL中传递的对象，必须实现Parcelable序列化接口；
* 在AIDL中传递的对象，需要在类文件相同路径下创建同名、后缀为.aidl的文件，并在文件中使用parcelable关键字声明这个类；
* AIDL接口跟普通接口的区别：只能声明方法，不能声明变量；
* 所有非基础数据类型参数都需要标出数据流向。可以是 in、out 或 inout，基础数据类型默认只能是 in，不能是其他方向。

&emsp;&emsp;AIDL只是提供给应用开发者使用的一种模板接口，本质上Android进程间的通信是通过Binder来实现的。至于什么是Binder、Binder是如何做到进程间通信的，这些问题涉及更深层次的探讨，本文不深入介绍，读者如果感兴趣的话，可以阅读本文末提供的参考资料入口。

**跨进程通信操作流程**
1. 定义AIDL跨进程通信接口及所需的Parcelable实体类；
2. 定义服务端进程Service，实现AIDL接口Stub内部类生成Binder对象；
3. 客户端使用bindService方法绑定服务端；
4. 服务端在onBind方法返回Binder对象；
5. 客户端拿到服务端返回的Binder对象进行跨进程方法调用；

![进程通信流程图](/assets/img/illustration/aidl_client_to_server.png)

#### AIDL实例
&emsp;&emsp;下面通过一个实例来讲解如何通过AIDL接口实现跨进程通信。在上一篇文章 [Android 多进程 -- 揭开神秘面纱]({{ site.baseurl }}{% post_url 2018-03-30-android-multi-process %}) 中我们提到了多进程推送服务的需求，这里就以推送服务作为我们的讲解实例。应用开发中，大多数时候后会涉及推送服务的需求，推送服务既是相对独立的模块又属于需要长时间占用后台的功能，因此使用私有子进程来保持推送服务是恰当的。

### 参考资料
* [Messenger 的工作原理](http://liwenkun.me/2017/02/25/how-does-messenger-work/)
* [Android：学习AIDL，这一篇文章就够了(上)](https://www.jianshu.com/p/a8e43ad5d7d2)
* [你真的理解AIDL中的in，out，inout么？](https://www.jianshu.com/p/ddbb40c7a251)
* [写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)
