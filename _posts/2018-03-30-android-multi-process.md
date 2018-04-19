---
layout: post
title: Android 多进程 -- 揭开神秘面纱
subtitle: That moment when one Dalvik alone is no longer enough.
tags: [Android, 多进程, Multi-Process]
---

&emsp;&emsp;Android是基于Linux系统开发的移动操作系统。进程系统也是一脉相承，进程，其实就是应用程序的具体实现。当应用程序第一次启动，Android会启动创建一个进程（由Zygote fork而来）以及一个主线程，默认的情况下，所有组件都将运行在该进程内。每个应用程序都在其自己的进程中运行（具有唯一的PID）：这允许应用程序运行在一个独立的环境中，不会受到其他应用程序/进程的影响。

### 为什么需要多进程
&emsp;&emsp;实际开发中，基本都不会去给app划分进程，而且，在Android中使用多进程，还可能需要编写额外的进程通讯代码，还可能带来额外的Bug，这无疑加大了开发的工作量。很多时候，业务开发量大，时间上也不允许，这导致了大多数的app都只有一个运行进程。

&emsp;&emsp;在Android中，虚拟机分配给各个进程的运行内存是有限制值的（这个值可能是32M，48M，64M等，根据机型而定）。试想一下，如果在app中，增加了一个很常用的图片选择模块用于上传图片或者头像，加载大量Bitmap会使app的内存占用迅速增加，如果你还把查看过的图片缓存在了内存中，OOM的风险将会大大增加。如果，此时app中还需要使用WebView加载一波网页...

&emsp;&emsp;主流应用是如何解决这些问题的？

&emsp;&emsp;微信开发团队曾在《Android内存优化杂谈》一文中提到：“对于webview，图库等，由于存在内存系统泄露或者占用内存过多的问题，我们可以采用单独的进程。微信当前也会把它们放在单独的tools进程中”。看看微信的进程分配情况：

![微信多进程](/assets/img/screenshots/wechat_multiporcess.png)

&emsp;&emsp;可以看到，微信的确有一个tools进程，还有其他多个进程，如com.tencent.mm:push。微信不单单只是把上述的WebView、图库等放到单独的进程，还有推送服务等也是运行在独立的进程中的。一个消息推送服务，为了保证稳定性，可能需要和UI进程分离，分离后即使UI进程退出、Crash或者出现内存消耗过高等情况，仍不影响消息推送服务。

&emsp;&emsp;**合理地使用多进程，可以增强应用程序的稳定性，降低OOM风险。**

### 实现方式
&emsp;&emsp;在AndroidManifest中注册四大组件Activity、Service、ContentProvider、BroadCastReceiver时设置process属性来指定它们运行在哪个进程中。指定进程后，当应用程序需要启动该组件时（如果进程不存在）便会先创建对应的进程，然后在该进程中运行组件。
```xml
<!-- 在系统组件中设置该属性 -->
android:process=":demo"
```

### 应用场景
&emsp;&emsp;总结一下，多进程的应用场景如下：
1. 单进程所分配的内存不够，需要更多的内存。
2. 某些相对独立的业务模块，使用单独的进程来运行。

### 优势
1. Android会给每个进程分配内存，可以通过设置子进程达到降低主进程内存占用目的；
2. Android在系统内存不足时不会同时Kill掉应用所有进程；

### 劣势
1. 单个应用开启多进程占用系统资源；
2. 某个进程保持后台长期运行，可能会导致耗电过多；
3. 应用程序的架构变复杂；
4. 需要考虑进程间的通信问题；

### 注意事项
1. 处理进程间通信；
2. Application初始化多次，需要针对不同进程初始化不同服务；
3. 以:开头的进程名字，表示这是一个应用程序的私有进程，否则它是一个全局进程；
4. 单例模式、静态变量等在不同进程会有不同的状态；

### IPC
&emsp;&emsp;常用的多进程通信（Inter-Process Communication）方式：
1. 四大组件间传递Bundle;
2. 使用文件共享方式，多进程读写一个相同的文件，获取文件内容进行交互；
3. 使用Messenger，一种轻量级的跨进程通讯方案，底层使用AIDL实现；
4. 使用AIDL(Android Interface Definition Language)，Android接口定义语言，用于定义跨进程通讯的接口；
5. 使用ContentProvider，常用于多进程共享数据，比如系统的相册，音乐等，我们也可以通过ContentProvider访问到；
6. 使用Socket传输数据。

&emsp;&emsp;四大组件传递Bundle，把需要传递的数据，用Intent封装起来传递即可。AIDL（Android Interface Definition Language）是Android提供给我们的标准跨进程通讯API，非常灵活且强大。Messenger是一种基于AIDL的轻量级IPC方案，在进程间传送Message对象从而实现进程间通讯。Message中可以传送Bundle对象，Bundle中可以传送我们实现了Parcelable接口的对象。使用Messenger不会出现并发读写问题，因为Messenger是以串行方式工作的，所以如果有大量的请求，不适合使用Messenger。

### 结语
&emsp;&emsp;类似图片选择这样的多进程需求，可能并不需要我们额外编写进程通讯的代码，使用四大组件传输Bundle就行了，但是像消息推送服务这种需求，进程与进程之间需要高度的交互，此时就绕不过进程通讯这一步了。后续将会探讨使用AIDL实现多进程消息推送服务的需求。
