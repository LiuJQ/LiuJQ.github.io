---
layout: post
title: Android 面经2018 -- Baidu
author: Jackin
tags: [Interview]
---

### View绘制流程
> onMeasure/onLayout/onDraw

### Activity#onNewIntent执行
> singleTask模式栈中已有实例再次调起，将会执行onNewIntent；

### Touch事件分发
> 1. onDispatchEvent/onInterceptTouceEvent/onTouchEvent
> 2. onDispatchEvent true/false事件如何分发

### 线程间通信
> 1. Handler实现线程通信，BroadCastReceiver/文件等；
> 2. Handler机制， Handler/Looper/MessageQueue；
> 3. MessageQueue中没有消息如何处理，Looper循环等待，可手动quit退出消息循环；

### IPC使用场景
> 1. 相对独立模块；
> 2. 需长时间后台服务；

### 同应用IPC方式
> 1. Messenger；
> 2. BroadCastReceiver；
> 3. Intent/Bundle；

### 布局优化
> 尽量减少layout嵌套层级，merge/include/ViewStub；

### 内存优化
> 1. Bitmap加载优化；
> 2. 减少内存泄漏问题；

### 内存泄漏
> 1. 内存泄漏场景，Listener/Runnable/View/Util未及时释放；
> 2. 内存泄漏原理，应用进程退出后gc发现有Context无法释放；
> 3. 如何分析内存泄漏点，工具使用AndroidStudio/MAT；
