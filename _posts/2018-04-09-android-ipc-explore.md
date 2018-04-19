---
layout: post
title: Android 多进程 -- 浅析IPC
subtitle: On Android, one process cannot normally access the memory of another process.
feature-img: "assets/img/pexels/old-phone.jpeg"
tags: [Android, 多进程, Multi-Process]
---

&emsp;&emsp;在前一篇文章 [Android 多进程 -- 揭开神秘面纱]({{ site.baseurl }}{% post_url 2018-03-30-android-multi-process %}) 中我们学习了如何在Android开发中使用多进程以及使用多进程需要注意的地方，那么伴随实现多进程而来的问题是，我们不得不解决跨进程之间的通信（IPC）问题。Android中解决IPC问题的方式多种多样，上一篇文章我们也总结过了，这次主要将介绍使用AIDL进行多进程通信，因为AIDL是Android提供给我们的标准跨进程通信API，非常灵活且强大。

&emsp;&emsp;跨进程通信的时候需要考虑一点，是否需要处理多线程并发的情况呢？

## Messenger
&emsp;&emsp;如果IPC不需要考虑多线程并发情况，那么Android提供了一种基于AIDL的轻量级IPC工具供开发者使用——Messenger。乍一看很熟悉，跟Handler处理的Message有关？对的，Messenger就是基于Handler传递Message的方式实现通信功能的，这也是为什么Messenger是线程安全（不支持并发）的原因。

&emsp;&emsp;查看源码不难发现Messenger是基于AIDL实现多进程通信的，Android SDK提供Messenger封装了Handler传递Message、然后通过IMessenger这个AIDL接口传递Message对象至Binder底层从而实现跨进程通信整个过程。

### IMessenger AIDL定义
&emsp;&emsp;文件位置：frameworks/base/core/java/android/os/IMessenger.aidl
```java
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
```

### Messenger IPC流程
&emsp;&emsp;*注：下图来源于[Messenger 的工作原理](http://liwenkun.me/2017/02/25/how-does-messenger-work/)，侵删。*

![How does Messenger work ?](/assets/img/illustration/how_does_messenger_work.png)

&emsp;&emsp;图中清晰地展示了由客户端到服务端的单向跨进程通信流程，可以看到，其实Message对象说到底也是通过AIDL接口传递到Binder再发送到其他进程的。很多同学可能以为Messenger的跨进程通信是通过Handler实现，其实这个理解有点本末倒置了。Handler只能处理处于同一进程空间的Message消息，跨进程的Message是通过Binder传递的。Messenger实现IPC的过程中，Handler只是扮演了在服务端（或者客户端）的一个Message消息分发者和消费者角色。

### Messenger如何使用
&emsp;&emsp;原理总是枯燥无味，代码更得人心。

&emsp;&emsp;下面我们通过代码来演示如何使用Messenger实现跨进程通信（为便于演示，Client与Server处于同一个Application）。

&emsp;&emsp;[MessengerDemo源码](https://github.com/LiuJQ/MessengerDemo)
#### 服务端
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

#### 客户端
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

## AIDL

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

**AIDL跨进程通信操作流程**
1. 定义AIDL跨进程通信接口及所需的Parcelable实体类；
2. 定义服务端进程Service，实现AIDL接口Stub内部类生成Binder对象；
3. 客户端使用bindService方法绑定服务端；
4. 服务端在onBind方法返回Binder对象；
5. 客户端拿到服务端返回的Binder对象进行跨进程方法调用；

![进程通信流程图](/assets/img/illustration/aidl_client_to_server.png)

### AIDL实例
&emsp;&emsp;下面通过一个实例来讲解如何通过AIDL接口实现跨进程通信。

&emsp;&emsp;[AIDLDemo源码](https://github.com/LiuJQ/AIDLDemo)

&emsp;&emsp;在上一篇文章 [Android 多进程 -- 揭开神秘面纱]({{ site.baseurl }}{% post_url 2018-03-30-android-multi-process %}) 中我们提到了多进程消息推送的需求，这里就以消息推送服务作为我们的讲解实例。应用开发中，有时候需要提供类似客服功能等涉及消息推送服务的需求，消息推送既是相对独立的模块又属于需要长时间占用后台的功能，因此使用私有子进程来保持消息推送服务是恰当的。

#### 需求分析
1. UI进程负责处理消息展示和发送操作；
2. 推送Service负责从远程拉取信息和发送UI进程传递的消息至远程；
3. 为保持消息的及时性，推送Service需与远程服务器保持长连接；
4. UI进程退出了，为收取消息，推送Service进程需保持后台运行；

#### 实现单向消息发送
* 定义消息主体

UI进程定义MessageModel实体Java类，不要忘记之前的注意事项，在AIDL中传递的非基础数据类型都必须实现Parcelable接口

```java
public class MessageModel implements Parcelable {
  private final static String MSG_FORMAT = "%s\nfrom:%s\nto:%s\nmessage:%s";

  private int msgId;
  private long msgTimeStamp;
  private String msgFrom;
  private String msgTo;
  private String msgContent;

  // empty constructor
  // getter and setter
  // parcelable methods

  @Override
  public String toString() {
      Date time = new Date(msgTimeStamp);
      return String.format(MSG_FORMAT, time.toString(), msgFrom, msgTo, msgContent);
  }
}
```
在相同包名的AIDL目录下创建MessageModel.aidl文件，并使用parcelable关键词声明

```java
// MessageModel.aidl
package com.jackin.aidldemo.model;

parcelable MessageModel;
```

* 定义AIDL消息发送接口

在AIDL目录下创建消息发送接口，不要忘记导入MessageModel类，并且在参数声明时定义好数据流向
```java
// IMessageSender.aidl
package com.jackin.aidldemo;

// Declare any non-default types here with import statements
import com.jackin.aidldemo.model.MessageModel;

interface IMessageSender {
    void sendMessage(in MessageModel msg);
}
```

* 私有进程定义Service，传递消息发送IBinder接口

```java
import com.jackin.aidldemo.IMessageSender;
import com.jackin.aidldemo.model.MessageModel;

public class RemoteMessageService extends Service {
    private final static String TAG = "RemoteMessageService";

    IBinder mMsgSender = new IMessageSender.Stub() {
        @Override
        public void sendMessage(MessageModel msg) throws RemoteException {
            // 私有进程Service接收到UI进程的消息
            Log.d(TAG, "send msg:\n" + msg.toString());

            // service should push this message to server
            // TODO 把UI进程发送的消息推送到远程服务器
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mMsgSender;
    }
}
```
在AndroidManifest文件中声明Service，并指定私有进程名称
```xml
<!-- 消息收发远程服务 -->
<service
    android:name=".remote.RemoteMessageService"
    android:process=":remote"
    android:exported="true"/>
```

* UI进程绑定Remote Service

```java
ServiceConnection mRemoteServiceConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        Log.i(TAG, "connected to remote service");
        mRemoteServiceBound = true;
        // 通过编译器自动生成的IMessageSender.java源码Stub内部类，转换得到AIDL定义的接口
        mMessageSender = IMessageSender.Stub.asInterface(service);
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {
        Log.i(TAG, "disconnected from remote service");
        mRemoteServiceBound = false;
    }
};

private void bindRemoteService() {
    Intent intent = new Intent(this, RemoteMessageService.class);
    bindService(intent, mRemoteServiceConnection, BIND_AUTO_CREATE);
}
```

* UI进程发起消息
```java
if (mRemoteServiceBound && mMessageSender != null) {
    // 创建消息体
    MessageModel msg = new MessageModel();
    msg.setMsgTimeStamp(System.currentTimeMillis());
    msg.setMsgFrom("Client");
    msg.setMsgTo("Service");
    msg.setMsgContent("What a good day today !");

    // 使用AIDL接口跨进程发送消息到Service
    try {
        mMessageSender.sendMessage(msg);
    } catch (RemoteException e) {
        Log.e(TAG, "send message failed");
        e.printStackTrace();
    }
} else {
    Log.e(TAG, "remote service disconnected.");
}
```

&emsp;&emsp;通过上述几个步骤就可以实现跨进程发送消息了，那么Service如何把从服务器接收到的消息传递给UI进程呢？

#### 实现双向消息传递

&emsp;&emsp;按照平时的开发经验，我们知道使用回调接口可以实现反向信息传递，那么是否可以应用到跨进程消息传递中呢？上面我们已经提到，AIDL接口的参数可以是其他AIDL接口，所以我们是可以通过**跨进程的回调接口**实现反向消息传递的。新增消息接收的AIDL接口IMessageReceiver.aidl，通过之前的消息发送接口IMessageSender.adil，在私有进程Service实现注册和反注册接收消息接口，Service收到消息时就可以利用IMessageReceiver接口传递到UI进程了。

[RemoteCallbackList](https://developer.android.com/reference/android/os/RemoteCallbackList.html)

```java
android.os.RemoteCallbackList<E extends android.os.IInterface>
```
&emsp;&emsp;说到跨进程的回调接口，不得不介绍一下RemoteCallbackList这个类。跨进程的回调接口总需要有一个容器来管理它们，普通的ArrayList在跨进程中可不管用了，这时候就要用RemoteCallbackList了。简单介绍RemoteCallbackList的几个特点：
* 内部作了多线程同步处理，线程安全
* 用于保持已注册的IInterface回调接口，可以准确地关联IInterface接口对应的IBinder对象
* 为每一个IInterface接口设置了[IBinder.DeathRecipient](https://developer.android.com/reference/android/os/IBinder.DeathRecipient.html)监听，自动清理已销毁进程对应的IInterface接口
* 遍历方式：beginBroadcast和finishBroadcast一定要配对使用

IMessageReceiver.aidl
```java
// IMessageReceiver.aidl
package com.jackin.aidldemo;

// Declare any non-default types here with import statements
import com.jackin.aidldemo.model.MessageModel;

interface IMessageReceiver {
    void onMessageReceived(in MessageModel msg);
}
```

IMessageSender.aidl新增跨进程回调接口的注册/反注册接口
```java
// IMessageSender.aidl
package com.jackin.aidldemo;

// Declare any non-default types here with import statements
import com.jackin.aidldemo.model.MessageModel;
import com.jackin.aidldemo.IMessageReceiver;

interface IMessageSender {
    void sendMessage(in MessageModel msg);

    void registerMessageReceiver(IMessageReceiver msgReceiver);

    void unRegisterMessageReceiver(IMessageReceiver msgReceiver);
}
```

UI进程绑定Service后注册消息接收回调接口
```java
@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    Log.i(TAG, "connected to remote service");
    mRemoteServiceBound = true;
    // 通过编译器自动生成的IMessageSender.java源码Stub内部类，转换得到AIDL定义的接口
    mMessageSender = IMessageSender.Stub.asInterface(service);
    try {
        // 注册消息接收回调接口
        mMessageSender.registerMessageReceiver(mMsgReceiver);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
```
UI进程销毁时主动反注册消息接收回调接口
```java
// 反注册接收消息回调接口
if (mMessageSender != null && mMessageSender.asBinder().isBinderAlive()) {
    try {
        mMessageSender.unRegisterMessageReceiver(mMsgReceiver);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
```

跨进程服务Service实现跨进程回调接口注册/反注册接口、模拟与服务器长连接
```java
public class RemoteMessageService extends Service {
    ...

    IBinder mMsgSender = new IMessageSender.Stub() {
        // sendMessage接口

        @Override
        public void registerMessageReceiver(IMessageReceiver msgReceiver) throws RemoteException {
            mMessageReceivers.register(msgReceiver);
        }

        @Override
        public void unRegisterMessageReceiver(IMessageReceiver msgReceiver) throws RemoteException {
            mMessageReceivers.unregister(msgReceiver);
        }
    };

    ...

    private class FakeMsgTCPTask implements Runnable {
        @Override
        public void run() {
            // 模拟长连接接收服务器推送的消息
            while (isServiceRunning.get()) {
                // 5s唤醒一次
                try {
                    Thread.sleep(1000 * 5);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                // TODO 从服务端拉取最新消息

                // 创建消息体
                MessageModel msg = new MessageModel();
                msg.setMsgFrom("Service");
                msg.setMsgTo("Client");
                msg.setMsgTimeStamp(System.currentTimeMillis());
                msg.setMsgContent("Someone say hello to you !");
                // 遍历所有跨进程连接
                final int receiverCount = mMessageReceivers.beginBroadcast();
                Log.d(TAG, receiverCount + " connection totally now");
                for (int index=0; index<receiverCount; index++) {
                    IMessageReceiver receiver = mMessageReceivers.getBroadcastItem(index);
                    if (receiver != null && receiver.asBinder().isBinderAlive()) {
                        try {
                          // 通过跨进程回调接口发送到UI进程
                            receiver.onMessageReceived(msg);
                        } catch (RemoteException e) {
                            e.printStackTrace();
                        }
                    } else {
                        Log.e(TAG, "binder died");
                    }
                }
                mMessageReceivers.finishBroadcast();
            }
        }
    }
}
```

&emsp;&emsp;以上就是利用AIDL实现跨进程双向交互的消息收发服务全部流程了，实际开发中我们可能还需要考虑跨进程连接的稳定性及安全性问题。关于稳定性问题，我们可以给跨进程设置DeathRecipient来监听跨进程连接，当连接断开时执行重新绑定跨进程操作。关于安全性问题，可以通过自定义Permission的方式，在跨进程绑定服务的时候判断UI进程是否已获得该权限来决定是否允许UI进程绑定。

&emsp;&emsp;由于篇幅过长，下次再通过另外一篇文章来详细介绍如何解决跨进程的稳定性和安全性问题。

## Demo源代码
* [MessengerDemo源码](https://github.com/LiuJQ/MessengerDemo)
* [AIDLDemo源码](https://github.com/LiuJQ/AIDLDemo)

## 参考资料
* [Messenger 的工作原理](http://liwenkun.me/2017/02/25/how-does-messenger-work/)
* [Android：学习AIDL，这一篇文章就够了(上)](https://www.jianshu.com/p/a8e43ad5d7d2)
* [你真的理解AIDL中的in，out，inout么？](https://www.jianshu.com/p/ddbb40c7a251)
* [写给 Android 应用工程师的 Binder 原理剖析](https://zhuanlan.zhihu.com/p/35519585)
