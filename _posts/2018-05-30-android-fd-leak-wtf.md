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
##### Thread

#### Input Channel File

### 案例分析

### 结语
