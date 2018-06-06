---
layout: post
title: Android Native Crash -- 浅谈及案例分析 FD 泄漏
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

&emsp;&emsp;Android 开发者对 Native Crash 问题可能并不陌生，但是谁也不愿意遇到它，毕竟此类问题十分棘手。此次我们要探讨的是由于 File Descriptor 泄漏导致的 Native Crash 。

&emsp;&emsp;File Descriptor 是一个索引值，指向内核为每个进程所维护的该进程打开文件的记录表。在 Unix/Linux 系统中，许多的资源都会被定义为 File Descriptor（下面简称FD），例如普通文件、socket、std in/out/error 等等。每个 Unix/Linux 系统中，单个进程可以使用的FD数量是有上限的。不同的 Unix/Linux 系统中，这个上限各有区别，例如在 Android 里面这个上限被限制为1024。一旦单个进程的FD数量超过上限时，Unix/Linux 系统就会kill掉这个进程，并且抛出 Native Crash。

### FD泄漏类型
&emsp;&emsp;在稳定性 Monkey 测试的过程中，经常会出现许多FD泄漏导致莫名其妙的FC，而 crash 的堆栈也是千奇百怪， 可能出现在应用层、framework 层、Native 层，其中以 framework 层居多。 所以当出现这个问题以后往往认为是 framework 出现问题了， 实际上从后面 Debug 的结果来看许多都是应用出现了问题。 同一个问题也会经常出现不同的堆栈，这就是FD泄漏的一个重要的特性，问题出现的不确定性。

#### Resource
&emsp;&emsp;Android 应用可能会需要很多资源，像输入输出流，数据库资源 Cursor， Binder 设备。如果没能够很好的处理这些资源，不仅可能造成内存的泄漏，也可能会出现FD泄漏。典型的错误日志如下：
> Parcel : dup() failed in Parcel::read, i is 0, fds[i] is -1, fd_count is 1, error: Too many open files
>
InputChannel-JNI: Error 24 dup channel fd 1023.

##### Stream
&emsp;&emsp;Android 中经常会使用到 FileInputStream，FileOutputStream，FileReader，FileWriter 等输入输出流，处理不好会导致内存溢出和FD泄漏。
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
&emsp;&emsp;线程的使用在 Android 中也是司空见惯，如果处理不好线程的释放，也会导致FD泄漏问题。
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

&emsp;&emsp;最近负责的应用在执行自动化测试的时候报了一个Native Crash，日志里面报出的原因也很明显，就是因为应用进程占用的FD句柄超出限制，系统强制kill掉了进程。所以排查的方向就按照上述提到的FD泄漏类型去推进。
> Long Msg: Native crash: Aborted
<br>signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
<br>Abort message: 'FORTIFY: FD_SET: file descriptor >= FD_SETSIZE'

&emsp;&emsp;分析案例前，先介绍几个有助于定位问题原因的文件。

_注：这些文件都是由测试平台提供的，具体如何抓取，不作详细介绍_

文件名称 | 文件抓取时机（测试平台提供） | 作用
:-----:|:-----:|:-----:
events_log | 整个Monkey过程 | 查询crash发生前程序执行了哪些操作
main_log | 整个Monkey过程 | 查询程序在最近几天的操作日志
logSnapshot | crash发生时抓取 | 查询crash发生时程序的内存状态、线程执行状态、crash堆栈信息以及设备和程序的基本信息
FD leaks文件 | crash发生时抓取 | 查询crash发生时程序的FD句柄情况

&emsp;&emsp;下面分几个步骤来分析此次出现FD泄漏导致Native Crash的案例。
#### Step 1 寻找crash页面
&emsp;&emsp;首先是查看events_log，看看crash发生前都执行了哪些操作。直接搜索"am_crash"可以定位到crash的操作位置，然后在往前翻一段日志可以查看到最近Monkey测试的操作。本次Native Crash我们发现是Monkey测试反复多次进入和退出Hybrid页面，然后出现了crash。

&emsp;&emsp;结合events_log的结果，再查看main_log可以跟踪到Native Crash发生前，Hybrid页面加载的是什么业务URL。

#### Step 2 确定FD泄漏类型
&emsp;&emsp;查看FD leaks列表文件可以帮我们快速定位是哪种FD泄漏类型。把FD leaks列表文件下载下来，然后统计FD数量排名前几的句柄，这里涉及一些Linux终端命令行的使用。
```xml
lrwx------ 1 root   root   64 2018-05-26 03:25 249 -> socket:[16451626]
lr-x------ 1 root   root   64 2018-05-26 06:49 25 -> /system/framework/WfdCommon.jar
lrwx------ 1 root   root   64 2018-05-26 00:55 250 -> /dev/ashmem
lrwx------ 1 root   root   64 2018-05-26 00:55 251 -> anon_inode:[eventpoll]
lrwx------ 1 root   root   64 2018-05-26 00:55 252 -> /dev/ashmem
lrwx------ 1 root   root   64 2018-05-26 03:14 253 -> anon_inode:[eventpoll]
```
&emsp;&emsp;使用awk命令来处理该文本，我们关心的是时间点和FD句柄的具体文件，因此我们可以以"64"分割，并打印出后面部分及保存到指定文件中。（这里假设FD列表源文件是FD_leaks.txt，处理后的目标文件是FD_leaks_dest.txt）
```bash
# cat命令可以输出文件内容，sort命令进行行排序，tee命令可以将过滤好的输出内容重定向保存至指定文件
cat FD_leaks.txt | awk -F 64 '{print $2}' | sort | tee FD_leaks_dest.txt
```
&emsp;&emsp;根据我们处理好的文件判断，目测发现有一个文件句柄出现的特别频繁，居然是本应用的base.apk文件。于是我们打算看看这个FD打开的次数（通过下面的Linux命令可以计算到，此处出现110次。）
```bash
# grep命令可以在指定文件中查找关键词，wc -l可以帮助我们快速计算行数
grep 'base.apk' FD_leaks_dest.txt | wc -l
110
```
&emsp;&emsp;程序在运行过程中多次打开了apk包中的某个文件没有释放！很明显，应该是某些二进制文件被引用后在特定条件（Monkey测试）下没有被释放。OK，我们可以直接解压apk包来看看res/raw文件夹下都有哪些文件。本应用下res/raw文件夹下有多个音频文件，猜测应该是某个音频文件在load完后未及时执行unload操作。

&emsp;&emsp;接下来我们可以查看logSnapshot文件，看看线程执行状态，如果是音频文件未及时释放，会出现SoundPool相关的线程。logSnap文件打印了出现FD泄漏crash时当前系统所有进程和线程信息，我们需要用本应用PID筛选出属于本应用进程的线程信息。以下是本应用进程（PID=6035）部分线程状态的信息：
```xml
u0_a39    13455 6035  2354260 410276 SyS_epoll_ 0000000000 S T_Sourc
u0_a39    13456 6035  2354260 410276 futex_wait 0000000000 S SoundPool
u0_a39    13457 6035  2354260 410276 futex_wait 0000000000 S SoundPoolThread
u0_a39    13458 6035  2354260 410276 futex_wait 0000000000 S SoundPool
u0_a39    13459 6035  2354260 410276 futex_wait 0000000000 S SoundPoolThread
```
&emsp;&emsp;利用这个线程状态信息文件，我们可以计算到SoundPool相关线程数量（此处恰好跟base.apk句柄数量惊人吻合，也印证了我们之前的猜测是正确的。）：
```bash
grep 'SoundPoolThread' fd_threads.txt | wc -l
110
```

#### Step 3 定位FD泄漏位置
&emsp;&emsp;结合上述两个步骤，我们可以发现是某个Hybrid页面，Monkey测试时触发了某含有音频播放的功能（该功能有音频操作漏洞，才会出现unload失败）导致SoundPoolThread释放不了，Monkey反复多次触发该功能则出现文件句柄不断增加，当应用进程句柄数量超过1024时系统底层Kill掉了应用进程并抛出异常信息。

&emsp;&emsp;在Android Studio中全局搜索工程目录，关键词是SoundPool。只有一个类ScrollTextView使用到，该类属于某个公共控件库，结合出现crash操作的Hybrid页面业务，我们定位到是选择日期的弹框控件使用到这个ScrollTextView控件（滚动选择日期时会播放跳动的声音）。

&emsp;&emsp;我们研究了一下ScrollTextView的源码，发现了一个漏洞，它内部初始化SoundPool的时候是用线程加载的，在线程内部用一个bool值来标记初始化完成状态，然后在release的时候去判断这个bool值为true才执行SoundPoll的unload操作。用户在正常时候的场景下，该流程可能不会出现问题，但仔细琢磨一下就知道，**当用户点击调起选择日期弹框控件快速退出，SoundPool初始化线程仍在执行中，但由于线程没有执行完毕，bool值变量依然为false，此时release操作是不会执行的，因此导致了SoundPool load完后却未执行unload操作而出现句柄泄露。**

&emsp;&emsp;猜测终究还需要实践来验证。OK，既然我们已经知道什么场景导致了此次FD泄漏，那么我们就来重现一下问题发生的过程。打开相应业务Hybrid页面，快速点击选择日期弹框，再快速退出，然后通过adb在命令行终端查看此时应用的FD状态（可以看到多次快速点击弹框退出后确实出现了SoundPoolThread逐渐增加，此处不再列出截图）：
```bash
# 通过以下命令来查看进程FD状态
adb shell ls -al /proc/${pid}/fd

# 通过以下命令来查看进程FD数量
adb shell ls -al /proc/${pid}/fd | wc -l
```

&emsp;&emsp;FD泄漏代码：
```java
public void initSoundPool(Context context) {
    this.mContext = context.getApplicationContext();
    this.mIsFinishedLoad = false;
    this.mSoundPoolThread = new Thread(new Runnable() {
        public void run() {
            if (VERSION.SDK_INT >= 21) {
                Builder builder = new Builder();
                builder.setMaxStreams(1);
                android.media.AudioAttributes.Builder attrBuilder = new android.media.AudioAttributes.Builder();
                attrBuilder.setLegacyStreamType(1);
                builder.setAudioAttributes(attrBuilder.build());
                SoudPoolHelper.this.mSoundPool = builder.build();
            } else {
                SoudPoolHelper.this.mSoundPool = new SoundPool(1, 1, 0);
            }

            SoudPoolHelper.this.mSoundPool.setOnLoadCompleteListener(new OnLoadCompleteListener() {
                public void onLoadComplete(SoundPool soundPool, int sampleId, int status) {
                    SoudPoolHelper.this.mIsFinishedLoad = true;
                }
            });
            SoudPoolHelper.this.mVoiceID = SoudPoolHelper.this.mSoundPool.load(SoudPoolHelper.this.mContext, raw.mc_picker_scrolled, 1);
            Looper.prepare();
            SoudPoolHelper.this.mSoundLooper = Looper.myLooper();
            SoudPoolHelper.this.mSoundHander = new Handler(SoudPoolHelper.this.mSoundLooper);
            Looper.loop();
        }
    });
    this.mSoundPoolThread.start();
}

public void release() {
    if (this.mIsFinishedLoad) {
        this.mSoundPool.unload(this.mVoiceID);
        this.mSoundPool.release();
        this.mSoundLooper.quit();
        this.mSoundPoolThread = null;
        this.mIsFinishedLoad = false;
        this.mContext = null;
    }
}
```

### 结语
&emsp;&emsp;发生FD泄漏的根本原因是没有对资源进行有效的管理。无论是文件资源、设备资源、Socket资源、输入输出流还是线程等，如果被频繁的调用而没有及时释放，甚至根本就没有释放，将会使得FD越积越多，最终导致了泄漏的发生。对照本文上述的几种可能导致FD泄漏的代码类型，逐步排查并修复，相信会对应用性能有所提升。

&emsp;&emsp;解决问题的关键还是得从源头开始。所以，在使用相关资源的时候脑子里知道应该做好后续的释放操作，才是有效避免问题发生的最好方法。毕竟，等到应用出现问题了再来寻找问题点，是一个非常痛苦的过程。
