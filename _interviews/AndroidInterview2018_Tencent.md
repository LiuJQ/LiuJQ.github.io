# Android 面经2018 -- 腾讯

### Hybrid文件上传速度优化
> 文件压缩，关注Webkit内核上传速度优化

### Java如何实现操作等待
> 线程同步问题，wait/notify

### Android内存泄漏
> 1. 什么情况会导致内存泄漏，Listener/Runable/View/Util；
> 2. 如何分析内存泄漏问题，用什么工具；
> 3. 内存泄漏原理，应用进程跑完后trim 80出发gc，发现仍有Context被持有；

### Android UI卡顿优化
> 1. 什么情况会导致UI卡顿；主线程执行耗时操作；
> 2. 谈谈实际开发中遇到的案例，并分析解决方法；

### Android IPC方式
> 1. 四大组件传递Bundle；
> 2. 共享文件；
> 3. BroadCastReceiver；
> 4. Mesenger实现简单的数据共享；
> 5. AIDL接口实现跨进程通信；
> 6. 网络；
> 7. ContentProvider；

### WebView加载URL的过程
> http请求相关的问题

### http协议3次握手过程
> 1. Client发起SYNC到Server，并设置状态为SEND；
> 2. Server收到SYNC后回复给Client，并设置状态为ANSWER；
> 3. Client收到Server的回复，并设置状态为ESTABLISHED；

### 加解密相关
> 1. 对称加密；
> 2. 不对称加密算法，加解密过程；

### 算法
> 1. 快速排序实现原理，分治递归思维；
> 2. 给定一个小范围随机数函数，要求提供计算大范围的随机数；

### 数据结构
> 如何判断一个链表有环，采取“快慢指针”方法，如果有环，在执行N次指针后移后两个指针肯定会相遇；

### 操作系统
> 1. 浮点数在操作系统中的存储；
> 2. for循环100万个数累加，如何优化，一次加4个数；
