### Activity启动模式及应用场景
> 1. standard 标准模式，每次启动都会创建实例放入栈中；
> 2. singleTop 栈顶复用模式，适用于通知消息拉起的页面，登录页面等；
> 3. singleTask 栈内复用模式，启动时clearTop，该实例到栈顶；通常是App的主入口；
> 4. singleInstance 全局单实例模式，独立栈，系统中只有一个该实例，适用于来电页面等；

### Activity#onNewIntent执行
> 1. 非standard模式的Activity再次被调起时会执行onNewIntent方法；
> 2. 回调onNewIntent方法时，传参Intent是最新的参数；
> 3. 回调onNewIntent方法时，需要调用setIntent方法才能更新Activity实例内部的Intent参数；

### Service的启动方式
> 1. startServie & bindService
> 2. 首次startServie会执行onCreate方法，再次startServie则执行onStartCommand；
> 3. bindService不调用onStartCommand，多次调用只会触发一次onBind方法；
> 4. 重复bindService/unbindService对多次回调onServiceConnected/onServiceDisconnected；
> 5. startServie/bindService同时启动，需要执行stopService/unbindService才可终止；

### BroadCastReceiver分类
> 1. 粘性/非粘性；
> 2. 有序广播/无序广播；
> 3. 全局广播/本地广播；
> 4. LocalBroadCastReceiver原理，Handler实现；

### Android应用启动（白屏）优化
> 1. Application尽量少操作；
> 2. MainActivity主线程尽量少耗时操作；
> 3. 延迟初始化必要服务，DecorView.postDelay；
> 4. 利用windowBackground属性，使得界面更快展现给用户；

### 内存泄漏
> 1. 内存泄漏场景，Listener/Runnable/View/Util未及时释放；
> 2. 内存泄漏原理，应用进程退出后gc发现有Context无法释放；
> 3. 如何分析内存泄漏点，工具使用LeakCannary/AndroidStudio/MAT；

### 线程和进程
> 1. 一个程序至少有一个进程,一个进程至少有一个线程
> 2. 进程在执行过程中拥有独立的内存单元，而多个线程共享内存，从而极大地提高了程序的运行效率
> 3. 进程是CPU运行的任务单元，线程是进程运行过程中的片段
> 4. 进程是资源分配的最小单位，线程是程序执行的最小单位

### IPC
> 1. 每一个进程启动时都会初始化一次Application，因此需要做好进程服务区分；
> 2. SharedPreference不能用于IPC，因为SP拥有自己的缓存机制，文件更新不及时导致多线程不安全；
> 3. SP文件在应用私有缓存目录，也不适用于跨应用的多进程。

> 应用场景
> > 1. 相对独立模块；
> 2. 需长时间后台运行的服务；

> 同应用IPC
> > 1. Messenger；
> 2. BroadCastReceiver；
> 3. Intent/Bundle；

> 跨应用IPC
> > 1. 四大组件传递Bundle；
> 2. 共享文件；
> 3. BroadCastReceiver；
> 4. Mesenger实现简单的数据共享；
> 5. AIDL接口实现跨进程通信；
> 6. 网络；
> 7. ContentProvider；

### 线程间通信
> 1. Handler实现线程通信，BroadCastReceiver/文件等；
> 2. Handler机制， Handler/Looper/MessageQueue；
> 3. MessageQueue中没有消息如何处理，Looper循环等待，可手动quit退出消息循环；

### 子线程更新UI
> 1. 只能在主线程中更新UI；
> 2. 在Activity#onCreate方法中创建线程操作UI可能不会报错，因为ViewRootImpl在onResume中才会被创建，而checkThread操作在ViewRootImpl中实现；

### 布局优化
> 1. include，复用布局文件；
> 2. merge，复用布局文件并减少层级嵌套；
> 3. ViewStub，按需加载布局；

### View绘制流程
> 1. onMeasure，View测量计算过程；
> 2. onLayout，View布局计算过程；
> 3. onDraw，View绘制过程；

### Touch事件分发
> Touch事件类型
> > MotionEvent.ACTION_DOWN<br>MotionEvent.ACTION_UP<br>MotionEvent.ACTION_MOVE<br>MotionEvent.ACTION_CANCEL

> Touch事件传递对象及流程
> > Activity -> ViewGroup -> View

> Touch事件分发过程及相关方法
> > 1. dispatchTouchEvent 点击事件能够传递给该View时，该方法会被调用；
> 2. onTouchEvent 处理点击事件，在onDispatchTouchEvent内部调用；
> 3. onInterceptTouchEvent ViewGroup独有方法，是否拦截某个事件，ViewGroup#dispatchTouchEvent内部调用；

### 数据结构
> 如何判断链表有环？
> > 快慢指针，一个指针每次走一步，另一个指针每次走两步，如果两个指针相遇，则链表有环

> 链表反转
> > 1. 利用栈将链表元素依次入栈，然后逐个出栈
> 2. 利用三个指针，两个用于反转，一个用于指向剩下的链表

> ArrayList/HashSet/LinkedList
> > 1. HashSet元素无序且不重复，ArrayList元素有序可重复
> 2. ArrayList轻量级，线程不安全，查询效率高，增删效率低；
> 3. LinkedList线程不安全，查询效率低，增删效率高；
> 4. ArrayList是基于动态数组的数据结构，LinkedList基于链表的数据结构

### Http、TCP/IP、UDP
> TCP/IP、UDP是传输层协议，Http是应用层数据封装协议

> TCP连接3次握手
> > 1. Client发起SYNC到Server，并设置状态为SYN_SEND；
> 2. Server收到SYNC后回复给Client，并设置状态为SYN_RECV；
> 3. Client收到Server的回复，并设置状态为ESTABLISHED；

> TCP和UDP区别
> > 1. TCP面向连接，可靠性高；
> 2. UDP面向报文，无连接，不可靠，传输速率高；

### HTTP URL请求过程
> 1. DNS域名解析；
> 2. TCP 3次握手；
> 3. 建立TCP连接后发起HTTP请求；
> 4. 服务器响应HTTP请求；
> 5. 浏览器解析HTML；
> 6. 浏览器渲染页面；

### sleep VS wait
> 1. sleep是Thread类定义的静态方法，wait是Object类定义的实例方法；
> 2. wait方法必须在synchronize块中使用，wait方法会释放锁；
> 3. sleep方法可以在线程中的任意位置使用，不释放锁；
> 4. wait方法必须配合notify方法使用，用于多线程操作；
> 5. sleep方法用于单线程休眠操作；

### SharedPreference
> 1. SP的get操作，会锁定SharedPreferences对象，互斥其他操作。
> 2. SP的put操作，getEditor及commitToMemory会锁定SharedPreferences对象，put操作会锁定Editor对象，写入磁盘更会锁定一个写入锁
>
> IO性能优化建议
> > 1. commit和apply的方法区别在于同步写入和异步写入，以及是否需要返回值。在不需要返回值的情况下，使用apply方法可以极大的提高性能。多个写入操作可以合并为一个commit/apply，将多个写入操作合并后也能提高IO性能。
> > 2. 由于锁的缘故，SP操作并发时，耗时会徒增。减少锁耗时，是另一个优化点。由于读写操作的锁均是针对SP实例对象的，将数据拆分到不同的sp文件中，便是减少锁耗时的直接方案。降低单文件访问频率，多文件均摊访问，以减少锁耗时。

### 普通for循环 VS 增强for循环
> 1. 普通for循环按照下标顺序遍历；
> 2. 增强for循环使用Iterator迭代遍历；
> 3. 增强for循环在使用过程中不能对元素进行修改，否则会导致ConcurrentModificationException；
> 4. 普通for循环更适用于数组类的数据结构遍历，增强for循环更适用于链表类数据结构遍历；

### String, StringBuffer, StringBuilder
> String的内部实现为final char[]，所以每次对现有的字符串修改的话都会新实例化一个String，所以频繁的字符串修改和拼接要用StringBuffer，在单线程环境下用StringBuilder，性能更好，因为可以避免加锁带来的性能损耗。

### Dalvik VS ART
> 1. Dalvik运行dex文件，支持即时编译，每次运行应用都需要将dex文件里的字节码转换成机器码；
> 2. ART(Android Runtime)应用在第一次安装的时候，字节码就会预先编译成机器码，使其成为真正的本地应用。这个过程叫做预编译（AOT,Ahead-Of-Time）。这样的话，应用的启动(首次)和执行都会变得更加快速。
>
> > ART优点:
> 1. 提高系统性能；
> 2. 提高应用启动速度；
>
> > ART缺点:
> 1. 机器码占用内存较高，导致应用安装后内存占用大；
> 2. 应用安装时间变长；

### 死锁
> 死锁的必要条件
> > 1. 互斥条件：一个资源每次只能被一个进程使用。
> > 2. 占有且等待：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
> > 3. 不可强行占有:进程已获得的资源，在末使用完之前，不能强行剥夺。
> > 4. 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。

### HTTP
> 公共头部
>
  字段 | 说明
  :---: | :---:
  Remote Address | 请求的远程地址
  Request URL | 请求的域名
  Request Method | 页面请求的方式，如GET/POST
  Status Code | 请求的返回状态
> Request Header
>
  字段 | 说明
  :---: | :---:
  Accept | 浏览器支持的 MIME 类型
  Accept-Encoding | 浏览器支持的压缩类型
  Accept-Language | 浏览器支持的语言类型
  Cache-Control | 指定请求和响应遵循的缓存机制
  Connection | 当浏览器与服务器通信时对于长连接如何进行处理：close/keep-alive
  Cookie | 向服务器返回cookie，这些cookie是之前服务器发给浏览器的
  Host | 请求的服务器URL
  Referer | 跳转页面的来源URL
  User-Agent | 客户端的一些必要信息
> Response Header
>
  字段 | 说明
  :---: | :---:
  Cache-Control | 告诉浏览器或者其他客户，什么环境可以安全地缓存文档
  Connection | 当client和server通信时对于长链接如何进行处理
  Content-Encoding | 数据在传输过程中所使用的压缩编码方式
  Content-Type | 数据的类型
  Date | 数据从服务器发送的时间
  Expires | 指定此次响应的有效期时间
  Set-Cookie | 设置和页面关联的cookie
  Transfer-Encoding | 数据传输的方式，chunked/identity

### HTTPS加密通信流程
> 1. 客户端向服务端发送请求
> 2. 服务端返回数字证书
> 3. 客户端用自己的CA[主流的CA机构证书一般都内置在各个主流浏览器中]公钥去解密证书,如果证书有问题会提示风险
> 4. 如果证书没问题客户端会生成一个对称加密的随机秘钥然后再和刚刚解密的服务器端的公钥对数据进行加密,然后发送给服务器端
> 5. 服务器端收到以后会用自己的私钥对客户端发来的对称秘钥进行解密
> 6. 之后双方就拿着这个对称加密秘钥来进行正常的通信
