---
layout: post
title: 浅谈及案例分析 Native Crash(file descriptor leaks)
subtitle: Android native crash because of file descriptor leaks
tags: [Android, fd泄漏, Native Crash]
---

> CRASH: com.process.name (pid 8088)
<br>Short Msg: Native crash
<br>Long Msg: Native crash: Aborted
<br>ABI: 'arm'
<br>pid: 8088, tid: 8088, name: com.process.name  >>> com.process.name <<<
<br>signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
<br>Abort message: 'FORTIFY: FD_SET: file descriptor >= FD_SETSIZE'

### 前言

&emsp;&emsp;Android 开发者对 Native Crash 问题可能并不陌生，但是谁也不愿意遇到它，毕竟此类问题十分棘手。此次我们要探讨的是由于 File Descriptor 泄露导致的 Native Crash 。

&emsp;&emsp;File Descriptor 是一个索引值，指向内核为每个进程所维护的该进程打开文件的记录表。在 Unix/Linux 系统中，许多的资源都会被定义为 File Descriptor（下面简称FD），例如普通文件、socket、std in/out/error 等等。每个 Unix/Linux 系统中，单个进程可以使用的FD数量是有上限的。不同的 Unix/Linux 系统中，这个上限各有区别，例如在 Android 里面这个上限被限制为1024。一旦单个进程的FD数量超过上限时，Unix/Linux 系统就会kill掉这个进程，并且抛出 Native Crash。

### FD泄露类型
&emsp;&emsp;在稳定性 Monkey 测试的过程中，经常会出现许多FD泄漏导致莫名其妙的FC，而 crash 的堆栈也是千奇百怪， 可能出现在应用层、framework 层、Native 层，其中以 framework 层居多。 所以当出现这个问题以后往往认为是 framework 出现问题了， 实际上从后面 Debug 的结果来看许多都是应用出现了问题。 同一个问题也会经常出现不同的堆栈，这就是FD泄漏的一个重要的特性，问题出现的不确定性。

#### Resource
&emsp;&emsp;Android 应用可能会需要很多资源，像输入输出流，数据库资源 Cursor， Binder 设备。如果没能够很好的处理这些资源，不仅可能造成内存的泄漏，也可能会出现FD泄漏。典型的错误日志如下：
> Parcel : dup() failed in Parcel::read, i is 0, fds[i] is -1, fd_count is 1, error: Too many open files
>
InputChannel-JNI: Error 24 dup channel fd 1023.

##### Stream
&emsp;&emsp;Android 中经常会使用到 FileInputStream，FileOutputStream，FileReader，FileWriter 等输入输出流，处理不好会导致内存溢出和FD泄露。
```java
String filename = prefix + "temp";
File file = new File(getCacheDir(), fileName);
try {
    file.createNewFile();  
    FileOutputStream out = new FileOutputStream(file);
} catch (FileNotFoundException e) {

} catch (IOException e) {

}
```
&emsp;&emsp;如果反复多次调用这段代码会出现FD随着调用次数而递增的问题，因为在使用完 FileOutputStream 后没有及时释放。每次创建 file 对象，即使它都是打开同一个文件，系统依然会每次都为进程创建不同的FD来指向这个文件流。正确的做法应该是在 finally 块中增加对 FileOutputStream 的 close 操作，这样即使在 try 块中发生了异常导致程序中断，FileOutputStream 依然能够得到释放。
```java
String filename = prefix + "temp";
File file = new File(getCacheDir(), fileName);
FileOutputStream out = null;
try {  
    file.createNewFile();  
    out = new FileOutputStream(file);  
} catch(Exception e) {  
} final {  
    if(out != null){  
      out.close();  
    }
}  
```
&emsp;&emsp;如果嫌上述写法麻烦，JDK 7 以后为我们提供了一种 [try-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html) 的简化写法，有兴趣的同学可以看看。
##### Cursor
&emsp;&emsp;与输入输出流相似，数据库查询的 Cursor 如果没有及时进行 Close 操作，也会出现FD泄漏的情况。
```java
public void problemMethod() {  
    Cursor cursor = query(); // 假设 query() 是一个查询数据库返回 Cursor 结果的函数   
    if (flag == false) {  // 出现了提前返回
        return;  
    }  
    cursor.close();  
}
```

#### Thread
&emsp;&emsp;线程的使用在 Android 中也是司空见惯，如果处理不好线程的释放，也会导致FD泄露问题。
##### HandlerThread
&emsp;&emsp;HandlerThread 是 Android 提供的异步任务处理线程类，结合 Handler 和 Runnable 使用可以实现大部分异步任务的处理工作。
```java
private void init() {
    mThread = new HandlerThread("background-refresh-thread");
    mThread.start();
    mHandler = new Handler(mThread.getLooper());
}

private void destroy() {
    if(mThread != null){
        mThread.quitSafely();
        mThread = null;
    }
    if(mHandler != null) {
        mHandler.removeCallbacksAndMessages(null);
        mHandler = null;
    }
}
```
&emsp;&emsp;上述的两个方法必须配套使用，否则会因为 Handler 中未处理完的消息回调造成内存泄漏以及 HandlerThread 没有及时释放导致FD泄漏。通常如果是在 Activity 中应用的话，在 onCreate 方法中调用 init, 在 onDestroy 方法中调用 destroy；如果是在 View 中使用的话，在 onAttachedToWindow 中调用 init，在 onDetachedFromWindow 中调用 destroy。
##### Thread
&emsp;&emsp;HandlerThread 实际上是带有 Looper 的 Thread，而对于传统的 Java Thread，需要声明 Looper 以后才会出现FD的增加。因为声明 Looper 相当于增加了一块缓冲区，需要有一个FD来标识。如果反复调用下面这段代码也会出现FD泄漏。如果确定不需要 Looper，可以使用 Looper.quit() 或者 Looper.quitSafely() 来退出 Looper，避免出现FD泄漏。
```java
Thread thread = new Thread (new Runnable() {  
    @Override  
    public void run() {  
        Looper.prepare();  
        // do things  
        Looper.loop();  
    }  
}).start();
```
#### Input Channel File
##### WindowManager.addView
&emsp;&emsp;WindowManager.addView 每次调用，都会在 server（WindowManagerService）和 Client(用户进程)端创建FD文件来作为 socket 通信，如果不调用 removeView 这个FD将得不到释放。事实上，如果 SystemServer 所在的进程的FD数量超过1024个，还会造成 Android 的重启。
##### Multi-Task
&emsp;&emsp;Activity 使用 Intent.FLAG_ACTIVITY_MULTIPLE_TASK 标识的时候，如果 Monkey 测试的时候该 Activity 被多次启动又没有及时销毁，则会导致FD泄漏问题。举个例子，写一封新邮件，startActivity 使用的 flag 是 multiTask，也就是说，每点击创建新的邮件都会创建task。而 Monkey 在跑的时候创建了n个邮件的 task， 而对应打开的 ComposeActivityEmail.java 的 “插入快速语” 会创建很多个fd，最终导致FD超过1024，进程崩溃。实际上，通过反复如下代码就会出现这个问题：
```java
Intent intent = new Intent();  
Intent.addFlags(Intent.FLAG_ACTIVITY_MULTIPLE_TASK);  
Intent.setClass(MainActivity.this, ComposeActivityEmail.class);  
startActivity(intent);
```
&emsp;&emsp;应用的 input event 由 WindowManagerService 管理，WMS内部会创建一个 InputManager，两者通过 InputChannel 来完成，WMS需要注册两个 InputChannel 与 InputManager 连接，其中 Server 端 InputChannel 注册在 InputManager（SystemServer），Client 端注册在应用程序主线程中。InputChannel 使用 Ashmem 匿名共享内存来传递数据，它由一个FD文件描述符指向，同时 read 端和 write 端各占用一个FD。创建一个新的 Task 时，server(system_server) 和 client(app) 都会构建FD。所以设置为Intent.FLAG_ACTIVITY_MULTIPLE_TASK 类似 flag 的时候，如果没有处理好 Activity 的生命周期，可能会出现 system_server 进程先于应用进程到达FD上限，造成 Android 系统重启。
### 案例分析

### 结语
