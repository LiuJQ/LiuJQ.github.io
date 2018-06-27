---
layout: post
title: Android 多线程 -- 异步处理技术概览
subtitle: Android asynchronous processing summary
tags: [Android, 多线程, 异步处理]
---

&emsp;&emsp;Java Thread 和 Android AsyncTask 是众所周知的异步处理方式，可以满足大部分场景的需求。但是，其实我们还可以有更多的选择，为不同的需求选择不同实现方式可以使我们的代码更稳定、更简洁和可读性更强。本文梳理了Android上的多种异步处理技术，让开发者更加清晰地了解这些异步处理技术的概念和用法，以及在不同的场景下应该如何选择合适的异步处理技术。

### AsyncTask
&emsp;&emsp;作为Android平台最为开发者熟知的异步处理API，AsyncTask易于实现并且提供了回调处理结果给主线程的接口。但AsyncTask也有一些无法避免的缺陷，比如无法感知Activity/Fragment的生命周期，所以当Activity/Fragment销毁时开发者必须主动处理AsyncTask的行为。这也就注定了AsyncTask不是后台耗时操作的最佳选择，一旦app在后台运行时被Android系统kill掉，异步任务也会同时被销毁。
```java
new AsyncTask<URL, Integer, Long>() {
     protected Long doInBackground(URL... urls) {
         int count = urls.length;
         long totalSize = 0;
         for (int i = 0; i < count; i++) {
             totalSize += Downloader.downloadFile(urls[i]);
             publishProgress((int) ((i / (float) count) * 100));
             // Escape early if cancel() is called
             if (isCancelled()) break;
         }
         return totalSize;
     }

     protected void onProgressUpdate(Integer... progress) {
         setProgressPercent(progress[0]);
     }

     protected void onPostExecute(Long result) {
         showDialog("Downloaded " + result + " bytes");
     }
}.execute(url1, url2, url3);
```

### IntentService
&emsp;&emsp;这是Android平台上长时间运行处理的合适选择，大文件的上传和下载就是一个很好的例子。即使用户退出了app上传和下载的操作依然可以继续执行，另外当这些任务在进行的时候也不会阻断用户继续操作app。

&emsp;&emsp;IntentService其实也不是什么黑科技，它的实现原理是通过继承Service在OnCreate时创建一个HandlerThread处理线程来执行耗时操作的，一旦任务处理完毕IntentService会调用stopSelf来自我销毁。
```java
public class RSSPullService extends IntentService {
    @Override
    protected void onHandleIntent(Intent workIntent) {
        // Gets data from the incoming Intent
        String dataString = workIntent.getDataString();
        ...
        // Do work here, based on the contents of dataString
        ...
    }
}
// AndroidManifest.xml
<application
    android:icon="@drawable/icon"
    android:label="@string/app_name">
    ...
    <!--
       Because android:exported is set to "false",
       the service is only available to this app.
    -->
    <service
        android:name=".RSSPullService"
        android:exported="false"/>
    ...
<application/>
```

### Loader
&emsp;&emsp;Loader是一个复杂的话题，任何单个Loader的实现都值得使用一篇文章来介绍。需要指出的是，Loader是Android 3.0(Honeycomb)以后提出的并且作为兼容库的一部分。Loader是可以感知Activity/Fragment的生命周期的，并且会缓存之前加载的数据。值得注意的是AsyncTaskLoader结合了AsyncTask和Loader的优点，解决了很多使用上的问题。

&emsp;&emsp;建议想深入了解Loader的开发者阅读[Google官方文档](https://www.androiddesignpatterns.com/2012/08/implementing-loaders.html)。

### JobScheduler
&emsp;&emsp;JobScheduler是在Android API 21以后推出的，至今未有Google官方的兼容库版本。使用JobScheduler稍微有点负责，幸运的是Google给出了使用样例[android-JobScheduler](https://github.com/googlesamples/android-JobScheduler/#readme)。实际上，需要创建一个Service和通过JobInfo.Builder来创建一个指定条件下启动Service的Job，这些条件可以是设备连接了无限量的网络或者设备正在充电等。当这些条件达到时并不保证Job一定会被执行，执行顺序也是不确定的，记住这一点很重要。

&emsp;&emsp;下面是从Google官方Sample里面提取出来的部分代码：
```java
JobInfo.Builder builder = new JobInfo.Builder(kJobId++, mServiceComponent);
String delay = mDelayEditText.getText().toString();
if (delay != null && !TextUtils.isEmpty(delay)) {
  builder.setMinimumLatency(Long.valueOf(delay) * 1000);
}
String deadline = mDeadlineEditText.getText().toString();
if (deadline != null && !TextUtils.isEmpty(deadline)) {
  builder.setOverrideDeadline(Long.valueOf(deadline) * 1000);
}
boolean requiresUnmetered = mWiFiConnectivityRadioButton.isChecked();
boolean requiresAnyConnectivity = mAnyConnectivityRadioButton.isChecked();
if (requiresUnmetered) {
  builder.setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED);
} else if (requiresAnyConnectivity) {
  builder.setRequiredNetworkType(JobInfo.NETWORK_TYPE_ANY);
}
builder.setRequiresDeviceIdle(mRequiresIdleCheckbox.isChecked());
builder.setRequiresCharging(mRequiresChargingCheckBox.isChecked());
mTestService.scheduleJob(builder.build());
// CANCEL JOBS
JobScheduler tm = (JobScheduler) getSystemService(Context.JOB_SCHEDULER_SERVICE);
tm.cancelAll();
```

### CountDownTimer
&emsp;&emsp;CountDownTimer没有特定的使用场景，开发者可以很便捷地使用它。需要注意的是，CountDownTimer是Context敏感的，所以当你退出Activity/Fragment时，记得Cancel掉CountDownTimer的实例，否则会造成内存泄露问题。声明一下，CountDownTimer实际上跟异步处理没有任何直接关系。如果你看它的[源码](https://github.com/aosp-mirror/platform_frameworks_base/blob/master/core/java/android/os/CountDownTimer.java)会发现，它就是通过Handler简单地postDelay一个message，这意味着它运行在你启动它的线程中，onTick()和onFinish()会在你启动CountDownTimer的线程中执行，所以你可以直接在这两个方法中执行UI操作。也是这个原因，在onTick()和onFinish()中不能执行繁重或耗时的操作。CountDownTimer被列在这里的原因是它的运行不会阻塞用户app操作即使它是在主线程被创建的，实际上这相当于异步的效果。另外请注意，如果你的更新interval很短，处理代码又耗时的话，可能阻塞运行线程。
```java
new CountDownTimer() {
    @Override
    public void onFinish() {
    }
    @Override
    public void onTick(long millisUntilFinished) {
    }
}.start();
```

### Java Threads or Android HandlerThread
&emsp;&emsp;Java Thread实现起来相当直截了当，但是开发者应该尽量避免在Android中使用。因为它运行中并不能直接更新UI，所以需要配合Handler来实现UI更新，所以AsyncTask会是更好的选择。另外线程启动后如果不加入一些控制代码，线程就是不受控的，可能会出现难以预期的问题。线程多次创建也会给系统带来很大的开销。
```java
new Thread(new Runnable(){
  public void run() {
    // do something here
  }
}).start();
```
&emsp;&emsp;Android HandlerThread, 可以处理后台线程的消息。由于消息处理倾向于做更多的重定向而不是处理，它的使用相当有限，但它为我们提供了一种在后台线程上执行一些任务的方法。一种使用场景就是在后台运行Service处理任务，IntentService就是这种原理。
```java
public class TickTockService extends Service {
  public void onCreate() {
      // Start up the thread running the service. Note that we create a
      // separate thread because the service normally runs in the process's
      // main thread, which we don't want to block. We also make it
      // background priority so CPU-intensive work will not disrupt our UI.
      mThread = new HandlerThread("ServiceArguments", Process.THREAD_PRIORITY_BACKGROUND);
      mThread.start();
      // Get the HandlerThread's Looper and use it for  our Handler
      mServiceLooper = mThread.getLooper();
      mServiceHandler = new Handler(mServiceLooper);
  }
}
```

### FutureTask
&emsp;&emsp;FutureTask可以执行异步处理任务，但是，如果处理结果没有准备好或者任务没有处理完成，调用get()方法将会阻塞线程。下面是一种使用方式，注意，确保不会阻塞UI线程。
```java
RequestFuture<JSONObject> future = RequestFuture.newFuture();
JsonObjectRequest request = new JsonObjectRequest(URL, null, future, future);
requestQueue.add(request);
```
&emsp;&emsp;可以通过future.get(30, TimeUnit.SECONDS)方式来阻塞线程，超过指定时间后会抛出超时异常，而不是无限期地等待。
```java
try {
  JSONObject response = future.get(); // this will block (forever)
} catch (InterruptedException e) {
  // exception handling
} catch (ExecutionException e) {
  // exception handling
}
```
### Java Timer / ScheduledThreadPoolExecutor
&emsp;&emsp;下面是一个使用Timer在5秒后执行任务的实例，它可以用来部署一个在后台线程运行的处理任务。但请记住，由于这种用法无法感知Activity/Fragment的生命周期，它内部任何对Activity/Fragment的强引用都可能会造成内存泄露问题。
```java
Timer timer = new Timer();
timer.schedule(new TimerTask(){
  public void run() {
    // time ran out.
    timer.cancel();
  }
}, 5000);
```
&emsp;&emsp;ThreadPoolExecutor可以延时或者定期执行一些给定的指令。当需要多个工作线程时，或者需要ThreadPoolExecutor（扩展类）的附加灵活性和兼容性时，它比Timer更受欢迎。按照先进先出(FIFO)的顺序启用完全相同执行时间的任务。
```java
public class CustomScheduledExecutor extends ScheduledThreadPoolExecutor {
   static class CustomTask<V> implements RunnableScheduledFuture<V> { ... }
   protected <V> RunnableScheduledFuture<V> decorateTask(
                Runnable r, RunnableScheduledFuture<V> task) {
       return new CustomTask<V>(r, task);
   }
   protected <V> RunnableScheduledFuture<V> decorateTask(
                Callable<V> c, RunnableScheduledFuture<V> task) {
       return new CustomTask<V>(c, task);
   }
   // ... add constructors, etc.
 }
// Code from Java API
```
&emsp;&emsp;ScheduledThreadPoolExecutor在Android上会有很多与Timer和Java Threads相同的问题。如果你想更新UI需要通过回调接口或者通过Handler发送消息到UI线程。ScheduledThreadPoolExecutor是Java API，无法感知Activity/Fragment的生命周期，开发者必须手动释放Listener接口以免造成内存泄露问题。

### 扩展
&emsp;&emsp;技术的更新迭代很快，近两年涌现出的RxJava大火，利用它在Android平台实现异步任务处理，可以使得代码更加简洁，可读性更高。当然新技术的学习有一定的成本，读者感兴趣可以关注相关文章：[给 Android 开发者的 RxJava 详解](https://gank.io/post/560e15be2dca930e00da1083)。
