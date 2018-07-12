# Android 面经2018 -- OPPO

### Android应用启动优化
> 1. Application尽量少操作；
> 2. MainActivity主线程尽量少耗时操作；
> 3. 延迟初始化必要服务，DecorateView.postDelay；

### 布局优化
> 1. 尽量减少layout嵌套层级，merge/include/ViewStub；
> 2. 多少层以上认为是过渡绘制；

### 内存泄漏
> 1. 什么情况会导致内存泄漏，Listener/Runable/View/Util；
> 2. 如何分析内存泄漏问题，用什么工具；

### 线程间通信
> 1. Handler/BoradCast/文件
> 2. Handler原理，简述Handler/Looper/MessageQueue；

### 进程间通信
> 1. 四大组件传递Bundle；
> 2. 共享文件；
> 3. BroadCastReceiver；
> 4. Mesenger实现简单的数据共享；
> 5. AIDL接口实现跨进程通信；
> 6. 网络；
> 7. ContentProvider；

### 设计模式
> 1. 简述熟悉的设计模式；
> 2. Android中的设计模式应用；

### Java
> 1. ArrayList和HashSet的区别；

### 数据库
> 1. Android实现哪个类，SQLiteOpenHelper；
> 2. 如何触发数据库升级，流程；
> 3. 事务的优缺点：
> > 1.可回滚；2.分批操作；3.减少IO操作；4.注意失败后回滚；

### BroadCastReceiver分类
> 1. 粘性/非粘性；
> 2. 有序广播/无序广播；
> 3. 全局广播/本地广播；
> 4. LocalBroadCastReceiver原理，Handler实现；

### Service
> 1. Service启动方式，startService/bindService；
> 2. 两种启动方式的区别；
> 3. 其他Service，如IntentService；

### Activity
> Activity生命周期；
> Activity启动模式；

### Handler机制
> Handler/Looper/MessageQueue

### 事件分发
> 1. onDispatchTouchEvent
> 2. onInterceptTouchEvent
> 3. onTouceEvent
