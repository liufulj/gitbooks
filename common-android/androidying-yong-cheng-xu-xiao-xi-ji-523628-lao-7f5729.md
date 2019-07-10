### **概述**

Android应用程序与传统的PC应用程序一样，都是消息驱动的。也就是说，在Android应用程序主线程中，所有函数都是在一个消息循环中执行的。Android应用程序其它线程，也可以像主线程一样，拥有消息循环。Android应用程序主线程是一个特殊的线程，因为它同时也是UI线程以及触摸屏、键盘等输入事件处理线程。主线程对消息循环很敏感，一旦发生阻塞，就会影响UI的流畅度，甚至发生ANR问题。  
主要讲Android应用程序线程消息循环原理，主要涉及到Handler和Looper两个类，以及根据消息循环的不同使用场景，总结出三种线程使用模型。掌握Android应用程序消息处理机制，有助于我们熟练地使用同步和异步编程，提高程序的运行性能。

* 线程与消息的关系
* 线程的消息队列创建
* 线程的消息循环
* 线程的消息发送
* 线程的消息处理
* 消息在异步任务的应用

#### **线程与消息的关系**

**Android应用程序有两种类型的线程**

* 带有消息队列，用来执行循环性任务
  * 有消息时就处理
  * 没有消息时就睡眠
  * 例子：主线程、android.os.HandlerThread
* 没有消息队列，用来执行一次性任务
  * 任务一旦执行完成便退出
  * 例子：java.lang.Thread

**带有消息队列的线程四要素**

* Message\(消息\)
* MessageQueue\(消息队列\)
* Looper\(消息循环\)
* Handler\(消息发送和处理\)

**Message、 MessageQueue、 Looper和Handler的交互过程**  
![](https://box.kancloud.cn/7dda2bebc9eaa4cde5c2fdfbeb96db49_546x357.png)

#### **线程的消息队列创建**

**MessageQueue与Looper的关系**  
![](https://box.kancloud.cn/95f870bb4606176a77c0938995e204f6_570x396.jpg)

**例1：ServerThread**  
![](https://box.kancloud.cn/7342d06e988dc673e3ee0a2502f1eeeb_615x531.png)

**例2：ActivityThread**  
![](https://box.kancloud.cn/754b52ab968f4ebb332f41d3b9889497_684x273.png)

**Looper.prepare/prepareMainLooper**  
![](https://box.kancloud.cn/bbcce720b5b5ea8526143f34f0f4884c_678x560.png)

**new Looper – 创建Java层的Looper**  
![](https://box.kancloud.cn/4f22905a83a0bb2618ea558bbf4e2615_465x178.png)

**new MessageQueue**  
![](https://box.kancloud.cn/15ea9d10a2bc49b18cb93d2854d9df84_436x343.png)

**nativeInit**  
![](https://box.kancloud.cn/14d740ea5553e3fec753a9368a6b8545_798x257.png)

**new NativeMessageQueue**  
![](https://box.kancloud.cn/f6bacc1c76082b6a39b66ea35caff497_770x122.png)

**new Looper – 创建C++层的Looper**  
![](https://box.kancloud.cn/fb26f3abc74bee75b69bbd3445b88193_868x377.png)

**pipe**

* 一种进程/线程间通信机制
* 包含一个写端文件描述符和一个读端文件描述符
* Looper通过读端文件描述符等待新消息的到来
* Handler通过写端文件描述符通知Looper新消息的到来

![](https://box.kancloud.cn/5342d2719cf0b10ceeb177f73d1a0677_557x188.png)

**epoll**

* 一种I/O多路复用技术，select/poll加强版
* epoll\_create：创建一个epoll句柄
* epoll\_ctl：设置要监控的文件描述符
* epoll\_wait：等待监控的文件描述符发生IO事件
* Looper利用epoll来监控消息队列是否有新的消息，也就是监控消息管道的读端文件描述符

**为什么要用epoll**  
Looper除了监控消息管道之外，还需要监控其它文件描述符，例如，用来接收键盘/触摸屏事件的文件描述符

#### **线程的消息循环**

**例1：ServerThread**  
![](https://box.kancloud.cn/7342d06e988dc673e3ee0a2502f1eeeb_615x531.png)

**例2：ActivityThread**  
![](https://box.kancloud.cn/754b52ab968f4ebb332f41d3b9889497_684x273.png)

**Looper.loop – Java层的Looper**  
![](https://box.kancloud.cn/c1c859edcda16d86b18abddd2234e5cb_694x480.png)

**MessageQueue.next**  
![](https://box.kancloud.cn/bb2bc78ebfbb1c3bf08f2a8481966f14_700x581.png)

**nativePollOnce**  
![](https://box.kancloud.cn/17e37c77f5d995c04a21396928e0eec4_721x83.png)

**NativeMessageQueue.pollOnce**  
![](https://box.kancloud.cn/695de9b609e935fd04ca338b8b2ee60c_559x93.png)

**Looper.pollOnce – C++层的Looper**  
![](https://box.kancloud.cn/841eb954c6e41f17775ed2fb80a9696f_779x360.png)

**Looper.pollInner**  
![](https://box.kancloud.cn/f3222f5cc7e3f1a06a4989898468f435_636x589.png)

**Looper.awoken**  
![](https://box.kancloud.cn/48c830fdf93c1c1ab7f53044372ab87b_667x157.png)

#### **线程的消息发送**

**常用的消息发送接口**

* Handler.sendMessage
 
  带一个Message参数，用来描述消息的内容
* Handler.post
 
  带一个Runnable参数，会被转换为一个Message参数

**Message**  
![](https://box.kancloud.cn/7938691aa1f1ae3ba78af8f3dd61c0f8_468x430.png)

**Handler.sendMessage/post**  
![](https://box.kancloud.cn/7399f96f0b7d8e24992a681481ac5302_524x395.png)

**Handler.sendMessageDelayed**  
![](https://box.kancloud.cn/8c461c0b1eae53363152e51f821e46d8_736x226.png)

**Handler.sendMessageAtTime**  
![](https://box.kancloud.cn/a729811d4ae643e874cf2561b4ae78e4_670x274.png)

**Handler.enqueueMessage**  
![](https://box.kancloud.cn/bcbdcb65bd8358a8614b0d4eb01d0450_809x225.png)

**MessageQueue.enqueueMessage**  
![](https://box.kancloud.cn/1edfff6915efae821dd2b71355942afe_585x589.png)  
**备注**：  
mBlocked：Indicates whether next\(\) is blocked waiting in pollOnce\(\) with a non-zero timeout.

**nativeWake**  
![](https://box.kancloud.cn/3875a6dd4f5499405e06c4537ffd931f_812x75.png)

**NativeMessageQueue.wake**  
![](https://box.kancloud.cn/e639027cc8f484306601d876bfc77afa_323x60.png)

**Looper.wake**  
![](https://box.kancloud.cn/c81efd87976c9776c5513cb3ad715651_609x242.png)

**回顾消息循环过程**  
![](https://box.kancloud.cn/454a5dbd4b6e3f74ca6eda72648324f4_731x391.jpg)

**Handler.dispatchMessage**  
![](https://box.kancloud.cn/dd49e39c634e90f2df7cd9226ca7c8b8_544x545.png)

#### **消息在异步任务的应用**

**在主线程为什么要用异步任务？**

* 主线程任务繁重
  * 执行组件生命周期函数
  * 执行业务逻辑
  * 执行用户交互
  * 执行UI渲染
* 主线程处理某一个消息时间过长时会产生ANR
  * Service生命周期函数 – 20s
  * Broadcast Receiver接收前台优先级广播函数 –10s
  * Broadcast Receiver接收后台优先级广播函数 – 60s
  * 影响输入事件处理的函数 – 5s
  * 影响进程启动的函数 – 10s
  * 影响Activity切换的函数– 2s

**备注**：  
一个进程如果正在接收一个前台优先级广播，那么它所在的进程调度组就为Process.THREAD\_GROUP\_DEFAULT ，如果是正在接收后台优先级广播，那么它所在的进程调度组就为Process.THREAD\_GROUP\_BG\_NONINTERACTIVE。发送广播时，通过Intent.FLAG\_RECEIVER\_FOREGROUND标志来指定是前台优先级还是后台优先级

**基于消息的异步任务接口**

* android.os.HandlerThread
 
  适合用来处于不需要更新UI的后台任务
* android.os.AyncTask
 
  适合用来处于需要更新UI的后台任务

**android.os.HandlerThread**  
![](https://box.kancloud.cn/737497aff95db032cf420f8dd6881be5_408x446.png)

**启动HandlerThread**

```
HandlerThread handlerThread 
=
new
HandlerThread
(
"Handler Thread"
)
;

handlerThread
.
start
(
)
;
```

**向HandlerThread分配任务**  
![](https://box.kancloud.cn/53849acb24072f6965c46a242537bd06_429x226.png)

```
ThreadTask threadTask 
=
new
ThreadTask
(
)
;

Handler handler 
=
new
Handler
(
handlerThread
.
getLooper
(
)
)
;


handler
.
post
(
threadTask
)
;
```

**退出HandlerThread**

```
handlerThread
.
quit
(
)
;
```

![](https://box.kancloud.cn/43259826815f83fa6abb3c248919e877_551x565.png)

**android.os.AyncTask**

* 在进程内维护一个线程池来执行任务
* 任务在开始执行、执行过程以及结束执行时均可以与主线程进行交互
* 任务是通过一个Handler向主线程发送消息以达到交互的目的

**例子**  
![](https://box.kancloud.cn/69d42672979ebfb66efb9278eb92cea1_692x576.png)

**AysnTask的相关成员变量定义**  
![](https://box.kancloud.cn/996ace050c8acf2bf74512a7adb12e92_763x465.png)

**用来向主线程发送消息的InternalHandler**  
![](https://box.kancloud.cn/70514de799f8bba23e0c0c6df2954ed3_581x404.png)

**任务执行过程或者结果数据--AsyncTaskResult**  
![](https://box.kancloud.cn/99d8101799cc2b0530216b78c853c71c_490x246.png)

**创建任务**  
![](https://box.kancloud.cn/03196c1ed0d95a86c923e6595c2d2332_714x578.png)

**执行任务**  
![](https://box.kancloud.cn/1454718f30ea25a390b919b23afc1630_653x231.png)  
![](https://box.kancloud.cn/ca7d4b763610502137514fdd97e4852d_681x508.jpg)

**触发AsyncTask.doInBackground在工作线程中被调用**

**更新任务进度**  
![](https://box.kancloud.cn/acc0f1f17dab5133528bb3f93054cbb9_625x160.jpg)  
![](https://box.kancloud.cn/d7a79190e26e571192c07133c3bfdcc6_651x409.jpg)  
**MESSAGE\_POST\_PROGRESS通过sHandler发送到主线程的消息队列**  
**触发AsyncTask.onProgressUpdate在主线程中被**

**任务结束时，MESSAGE\_POST\_RESULT通过sHandler发送到主线程的消息队列，触发AsyncTask.finish在主线程中被调用**  
![](https://box.kancloud.cn/61b2ad372eb4f871bb52bc36630d6d65_482x176.png)  
![](https://box.kancloud.cn/aabaed00cb052ee3b38dd40eb7b97bc5_679x487.jpg)  
![](https://box.kancloud.cn/dcddd565ec263b06024d5b64e1b3a8e5_657x410.jpg)

**进一步触发AsyncTask.onPostExecute在主线程中被调用**

**任务中途取消时，MESSAGE\_POST\_CANCEL通过sHandler发送到主线程的消息队列，触发AsyncTask.onCancel在主线程中被调用**  
![](https://box.kancloud.cn/4dd0c6ad9e541028fd9d97d6c18c2c4f_504x131.png)  
![](https://box.kancloud.cn/182ea492e0898d3e823cf69333403000_691x514.jpg)  
![](https://box.kancloud.cn/f48e612e75149ecf3241aabc21613826_668x418.jpg)

