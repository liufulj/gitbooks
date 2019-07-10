### **概述**

Android专用驱动构成了Android运行时的基石。从技术上来讲，Android专用驱动也是整个Android系统的亮点，特别是Binder驱动。Binder是一种进程间通信机制\(IPC\)，它与传统的IPC机制对比，最大的特点是高效，因为通信数据在两个进程之间只需要执行一次拷贝即可。Binder在Android系统里面使用得非常广泛以及频繁。在涉及到比较大的通信数据时，Binder通常还结合另外一个驱动Ashmem来使用。Ashmem是一个共享内存驱动，它与传统的共享内存相比，最大的特点是它是通过文件描述符来描述的，并且可以动态地进行分块管理。动态分块管理的目的是可以将部分不再使用了的内存交回给系统，非常适合内存较小的移动设备使用。另外一个专用驱动Logger是一个日志驱动，它与传统的日志系统对比，特点是日志是记录在内核空间而非文件中，这样就可以提高日志的读写速度。  
主要讲Logger、Binder和Ashmem三个Android专用驱动的实现原理。由于这三个驱动在Android源代码里面用得非常广泛和频繁，因此理解它们的实现原理，就可以掌握Android的精华。这对以后阅读Android系统的其它代码，也是非常有帮助的。

* Android专用驱动概述
* Android Logger驱动系统
* Android Binder驱动系统
* Android Ashmem驱动系统

#### **Android专用驱动概述**

**以Linux驱动形式实现在内核空间**

* 不是为了驱动硬件设备工作
* 而是为了获得特权管理系统

**为Android Runtime Framework服务**

* 高效的日志服务
* 高效的进程间通信机制
* 以组件为目标的进程管理机制
* ……

**在整个系统中广泛和频繁地使用**

#### **Android Logger驱动系统**

**日志系统的作用和特点**

* 开发期间调试程序功能
* 发布期间记录程序运行
* 频繁和广泛地使用

**传统日志系统以文件为输出**

* 从用户空间切换至内核空间
* 在内核空间执行磁盘IO操作

**Android日志系统以内核缓冲区为输出**

* 从用户空间切换至内核空间
* 直接内存操作

**Android日志系统更高效**

**以缓冲区为输出的日志系统要解决的问题**

* 不能占用过多的内存
* 次要日志不能覆盖重要日志
* 频繁写入的日志不能覆盖不频繁写入的日志

**以上三个问题的解决方案**

* 环形缓冲区，新日志覆盖旧日志
* 日志分类，不同类的日志写在不同的缓冲区

**整体架构图**  
![](https://box.kancloud.cn/e60e2154623e98e9a761142571a7efd3_573x506.jpg)

**日志分类**

* Main，记录应用程序类型日志，次要，文本格式
 
  /dev/log/main
* System，记录系统类型日志，重要，文本格式
 
  /dev/log/system
* Radio，记录无线相关日志，频繁 ，文本格式
 
  /dev/log/radio
* Event，记录系统事件日志，特殊用途，二进制格式
 
  /dev/log/events

**每一类日志对应一个设备文件**

* Main：/dev/log/main
* System： /dev/log/system
* Radio：/dev/log/radio
* Event：/dev/log/events

**每一类日志对应一个环形缓冲区，大小256K**

**备注**  
在Linux Kernel 2.6.29的实现中：

1. 不区分Main和System类型的日志，它们对应的都是同一个环形缓冲区
2. Main、System和Event类型的日志对应的环形缓冲区大小为64K，Radio类型的日志对应的环形缓冲区大小256K

在Linux Kernel 3.4的实现中：

1. Main和System类型已经作区分，它们对应的是不同环形缓冲区
2. 所有环形缓冲区的大小均为256K

**文本类型日志格式**  
![](https://box.kancloud.cn/b28b44357f90e8cd4c03da164f3d8665_767x37.jpg)

**Priority，优先级，整数**  
VERBOSE、DEBUG、INFO、WARN、ERROR、FATAL

**Tag，标签，字符串**  
自定义

**Msg，日志内容，字符串**  
自定义

**二进制类型日志格式**  
![](https://box.kancloud.cn/ae10abd85641c4e4852a516c998a1a27_781x39.jpg)

**Tag，标签，整数**  
自定义  
**Msg，日志内容，二进制数据块**  
![](https://box.kancloud.cn/ccdbf3e81d8a2fe87e68e65cd3e31047_782x43.jpg)

**Type：整数\(1\)，长整数\(2\)，字符串\(3\)，列表\(4\)**  
**Value：自定义**

**二进制日志格式由/system/etc/event-log-tags文件描述**  
![](https://box.kancloud.cn/0e29ce550f4976bd94409fcc787bd8f8_820x39.jpg)

* 日志标签tag number对应的文本描述为tag name
* 日志内容格式
 
  ![](https://box.kancloud.cn/b70e3f053d0b4c18f24ee3527870bd68_810x35.jpg)
  * name：名称
  * data type：数据类型
  * data unit：数据单位

**二进制日志格式描述示例**

```
2722
 battery_level 
(
level
|
1
|
6
)
,
(
voltage
|
1
|
1
)
,
(
temperature
|
1
|
1
)
```

* 日志2722表示日志标签值，battery\_level是其对应的文件描述
* 标签值等于2722的日志内容由三个值组成，它们分别是level、voltage和temperature
* level 、voltage和temperature的数据类型均是整数\(1\)
* level的单位是百分比\(6\)
* voltage和temperature的单位均为对象数量\(1\)

**用户空间日志库**  
![](https://box.kancloud.cn/953886ca2a98960cf38f06c6ed64dbef_679x346.jpg)

* \_\_android\_log\_assert、\_\_android\_log\_vprint、\_\_android\_log\_print
 
  写入类型为main的日志记录
* \_\_android\_log\_btwrite、\_\_android\_log\_bwrite
 
  写入类型为event的日志记录
* \_\_android\_log\_buf\_print
 
  写入任意类型的日志记录
* \_\_android\_log\_write、\_\_android\_log\_buf\_write
 
  日志标签以“RIL”开头或者于“HTC\_RIL”、“AT”、“GSM”、“STK”、“CDMA”、“PHONE”或“SMS”，那么会被认为是radio类型的日志记录
* write\_to\_log
  * 函数指针，负责最终的日志写入
  * 开始时指向\_\_write\_to\_log\_init
  * 初始化成功后指向\_\_write\_to\_log\_kernel
  * 初始化失败后指向\_\_write\_to\_log\_null

**C/C++日志写入接口**

* LOGV、LOGD、LOGI、LOGW、LOGE
* SLOGV、SLOGD、SLOGI、SLOGW、SLOGE
* LOG\_EVENT\_INT、LOG\_EVENT\_LONG、LOG\_EVENT\_STRING

**Java日志写入接口**

* android.util.Log
* android.util.Slog
* android.util.EventLog

**日志读取工具-- logcat**

```
adb logcat 
--
help

```

![](https://box.kancloud.cn/a7b61897f753f913181d8d8e818d6542_610x362.jpg)

#### **Android Binder驱动系统**

**传统的IPC ，例如Pipe和Socket，执行一次通信需要两次数据拷贝**  
![](https://box.kancloud.cn/8b4d6eb58b49430f33dbf3890bac6f1b_474x198.jpg)  
**内存共享机制虽然只需要执行一次数据拷贝，但是它需要结合其它IPC来做进程同步，效率同样不理想**

**Binder是一种高效且易用的IPC机制**

* 一次数据拷贝
* Client/Server通信模型
* 既可用作进程间通信，也可用作进程内通信
 
  ![](https://box.kancloud.cn/8fdad34947e3b3b60716ea034beef2e9_784x465.jpg)

**Binder驱动为每一个进程分配4M的内核缓冲区，用作数据传输**

* 4M内核缓冲区所对应的物理页面是按需要分配的，一开始只有一个物理页被被映射
* 4M内核缓冲区所对应的物理页面除了映射在内核空间之外，还会被映射在进程的用户空间
 
  ![](https://box.kancloud.cn/5310f756866da0d25e1b1e09dfcf6f2e_890x280.jpg)

**进程间的一次数据拷贝**  
![](https://box.kancloud.cn/0c8a7104ffa359c822daf24324c12584_441x229.jpg)

**Client/Server通信模型**

* Server进程启动时，将在本进程内运行的Service注册到Service Manager中，并且启动一个Binder线程池，用来接收Client进程请求
* Client进程向Service Manager查询所需要的Service，并且获得一个Binder代理对象，通过该代理对象即可向Service发送请求

**Service注册**

* BBinder：Service在用户空间的描述
* binder\_node：Service在内核空间的描述
* binder\_ref：Service代理在内核空间的描述\(一个整数句柄值\)
 
  ![](https://box.kancloud.cn/36fc4f70aba9b5ba47c59c50cd86b285_527x360.jpg)

**Service代理获取**

* BpBinder：Service代理在用户空间的描述\(一个整数句柄值\)
 
  ![](https://box.kancloud.cn/67bd9312adeea9e56cb55ce145937ca8_764x360.jpg)

**Service Manager注册及其代理获得**  
一个特殊Service，它的代理句柄值永远等于0  
![](https://box.kancloud.cn/1bad4cadb748d7bf946baad884de1275_680x355.jpg)

**Client和Server的通信过程**  
![](https://box.kancloud.cn/a132f9101f37fcaea431846694d94638_589x385.jpg)

**binder\_transaction：通信数据描述**  
![](https://box.kancloud.cn/667407d4a1c6bc1aa631496fe17958fb_517x329.png)  
![](https://box.kancloud.cn/e1d030d0cb30ca22701961b370e5e077_610x285.png)

**Binder对象\(flat\_binder\_object\)的类型**

* BINDER\_TYPE\_BINDER
* BINDER\_TYPE\_WEAK\_BINDER
* BINDER\_TYPE\_HANDLE
* BINDER\_TYPE\_WEAK\_HANDLE
* BINDER\_TYPE\_FD

![](https://box.kancloud.cn/f1fdb489e766c3868119587799954c15_866x245.jpg)

**BINDER\_TYPE\_BINDER和BINDER\_TYPE\_WEAK\_BINDER类型的flat\_binder\_object传输发生在**：

* Server进程主动向Client进程发送Service \(匿名Service \)
* Server进程向Service Manager进程注册Service
 
  **BINDER\_TYPE\_HANDLE和BINDER\_TYPE\_WEAK\_HANDLE类型的flat\_binder\_object传输发生在**
  ：
 
  一个Client向另外一个进程发送Service代理
 
  **BINDER\_TYPE\_FD类型的flat\_binder\_object传输发生在**
  ：
 
  一个进程向另外一个进程发送文件描述符

**Binder驱动对类型BINDER\_TYPE\_BINDER 的Binder对象\(flat\_binder\_object\)的处理**：

* 在源进程中找到对应的binder\_node。如果不存在，则创建。
* 根据上述binder\_node在目标进程中找到对应的binder\_ref。如果不存在，则创建。
* 增加上述binder\_ref的强引用计数和弱引用计数
* 构造一个类型为BINDER\_TYPE\_HANDLE的flat\_binder\_object对象。
* 将上述flat\_binder\_object对象发送给目标进程。

**Binder驱动对类型BINDER\_TYPE\_WEAK\_BINDER 的Binder对象\(flat\_binder\_object\)的处理**：

* 在源进程中找到对应的binder\_node。如果不存在，则创建。
* 根据上述binder\_node在目标进程中找到对应的binder\_ref。如果不存在，则创建。
* 增加上述binder\_ref的弱引用计数。
* 构造一个类型为BINDER\_TYPE\_WEAK\_HANDLE的flat\_binder\_object对象。
* 将上述flat\_binder\_object对象发送给目标进程。

**Binder驱动对类型BINDER\_TYPE\_WEAK\_BINDER 的Binder对象\(flat\_binder\_object\)的处理**：

* 在源进程中找到对应的binder\_node。如果不存在，则创建。
* 根据上述binder\_node在目标进程中找到对应的binder\_ref。如果不存在，则创建。
* 增加上述binder\_ref的弱引用计数。
* 构造一个类型为BINDER\_TYPE\_WEAK\_HANDLE的flat\_binder\_object对象。
* 将上述flat\_binder\_object对象发送给目标进程。

**Binder驱动对类型BINDER\_TYPE\_HANDLE 的Binder对象\(flat\_binder\_object\)的处理**：

* 在源进程中找到对应的binder\_ref。
  * 如果上述binder\_ref所引用的binder\_node所在进程就是目标进程：
    * 增加上述binder\_node的强引用计数和弱引用计数
    * 构造一个类型为BINDER\_TYPE\_BINDER的flat\_binder\_object
    * 将上述flat\_binder\_object发送给目标进程
  * 如果上述binder\_ref所引用的binder\_node所在进程不是目标进程：
    * 为目标进程创建一个binder\_ref，该binder\_ref与上述binder\_ref引用的是同一个binder\_node
    * 增加上述新创建的binder\_ref的强引用计数和弱引用计数
    * 构造一个类型为BINDER\_TYPE\_HANDLE的flat\_binder\_object
    * 将上述flat\_binder\_object发送给目标进程

**Binder驱动对类型BINDER\_TYPE\_WEAK\_HANDLE 的Binder对象\(flat\_binder\_object\)的处理**：

* 在源进程中找到对应的binder\_ref。
  * 如果上述binder\_ref所引用的binder\_node所在进程就是目标进程：
    * 增加上述binder\_node的弱引用计数
    * 构造一个类型为BINDER\_TYPE\_WEAK\_BINDER的flat\_binder\_object
    * 将上述flat\_binder\_object发送给目标进程
  * 如果上述binder\_ref所引用的binder\_node所在进程不是目标进程：
    * 为目标进程创建一个binder\_ref，该binder\_ref与上述binder\_ref引用的是同一个binder\_node
    * 增加上述新创建的binder\_ref的弱引用计数
    * 构造一个类型为BINDER\_TYPE\_WEAK\_HANDLE的flat\_binder\_object
    * 将上述flat\_binder\_object发送给目标进程

**Binder驱动对类型BINDER\_TYPE\_FD 的Binder对象\(flat\_binder\_object\)的处理**：

* 在源进程中找到对应的struct file结构体
* 将上述struct file结构体 保存在目标进程的打开文件列表中
* 构造一个类型为BINDER\_TYPE\_FD的flat\_binder\_object
* 将上述flat\_binder\_object发送给目标进程

**Binder对象的引用计数**

* BBinder：位于用户空间，通过智能指针管理，有mStrong和mWeak两个引用计数
* BpBinder：位于用户空间，通过智能指针管理，有mStrong和mWeak两个引用计数
* binder\_node：位于内核空间，有internal\_strong\_refs、local\_weak\_refs和local\_strong\_refs三个引用计数，以及一个binder\_ref引用列表refs
* binder\_ref：位于内核空间，有strong和weak两个引用计数

**备注**：  
local\_strong\_refs和local\_weak\_refs分别表示该Binder实体对象的内部强引用计数和弱引用计数；而internal\_strong\_refs是一个外部强引用计数，用来描述有多少个Binder引用对象是通过强引用计数来引用该Binder实体对象的。内部引用计数是相对于该Binder实体对象所在的Server进程而言的，而外部引用计数是相对于引用了该Binder实体对象的Client进程而言的。当Binder驱动程序请求Server进程中的某一个Binder本地对象来执行一个操作时，它就会增加与该Binder本地对象对应的Binder实体对象的内部引用计数，避免该Binder实体对象被过早地销毁；而当一个Client进程通过一个Binder引用对象来引用一个Binder实体对象时，Binder驱动程序就会增加它的外部引用计数，也是避免该Binder实体对象被过早地销毁。读者可能会觉得奇怪，为什么一个Binder实体对象的外部引用计数只有强引用计数internal\_strong\_refs，而没有一个对应的弱引用计数internal\_weak\_refs呢？原来，Binder实体对象的Binder引用对象列表refs的大小就已经隐含了它的外部弱引用计数，因此，就不需要使用一个额外的弱引用计数internal\_weak\_refs来描述它的外部弱引用计数了。

**Binder对象之间的引用关系**  
![](https://box.kancloud.cn/433a78d86017bc52ecc26907b2cde9dd_587x96.png)![](https://box.kancloud.cn/cbd6d402789894cf923d97d684af0420_470x546.jpg)

**Binder对象引用关系小结**

* BBinder被binder\_node引用

* binder\_node被binder\_ref引用

* binder\_ref被BpBinder引用

* BBinder和BpBinder运行在用户空间

* binder\_node和binder\_ref运行在内核空间

* 内核空间足够健壮，保证binder\_node和binder\_ref不会异常销毁

* 用户空间不够健壮，BBinder和BpBinder可能会异常销毁

* BpBinder异常销毁不会引发致命问题，但是BBinder异常销毁会引发致命问题

* 需要有一种方式来监控BBinder和BpBinder的异常销毁

**Binder对象异常销毁监控**

* 所有执行Binder IPC的进程都需要打开/dev/binder文件

* 进程异常退出的时候，内核保证会释放未正常关闭的它打开的/dev/binder文件，即调用与/dev/binder文件所关联的release回调函数

* Binder驱动通过实现/dev/binder文件的release回调函数即可监控Binder对象的异常销毁，进而执行清理工作

* BBinder异常销毁的时候，不单止Binder驱动需要执行清理工作，引用了它的BpBinder所在的Client进程也需要执行清理工作

* 需要有一种BBinder死亡通知机制

  * Client进程从Binder驱动获得一个BpBinder
  * Client进程向Binder驱动注册一个死亡通知，该死亡通知与上述BpBinder所引用的BBinder相关联
  * Binder驱动监控到BBinder所在进程异常退出的时候，检查该BBinder是否注册有死亡通知
  * Binder驱动向注册的死亡通知所关联的BpBinder所运行在的Client进程发送通知
  * Client进程执行相应的清理工作

**Binder线程池**

* 在Server进程中，Client进程发送过来的Binder请求由Binder线程进行处理
* 每一个Server进程在启动的时候都会创建一个Binder线程池，并且向里面注册一个Binder线程
* 之后Server进程可以无限地向Binder线程池注册新的Binder线程
* Binder驱动发现Server进程没有空间的Binder线程时，会主动向Server进程请求注册新的Binder线程
* Binder驱动主动请求Server进程注册新的Binder线程的数量可以由Server进程设置，默认是16

**Binder线程调度机制**

* 在Binder驱动中，每一个Server进程都有一个todo list，用来保存Client进程发送过来的请求，这些请求可以由其Binder线程池中的任意一个空闲线程处理
* 在Binder驱动中，每一个Binder线程也有一个todo list，用来保存Client进程发送过来的请求，这些请求只可以由该Binder线程处理
* Binder线程没事做的时候，就睡眠在Binder驱动中，直至它所属的Server进程的todo list或者它自己的todo list有新的请求为止
* 每当Binder驱动将Client进程发送过来的请求保存在Server进程的todo list时，都会唤醒其Binder线程池的空闲Binder线程池，并且让其中一个来处理
* 每当Binder驱动将Client进程发送过来的请求保存在Binder线程的todo list时，都会将其唤醒来处理

**默认情况下， Client进程发送过来的请求都是保存在Server 进程的todo list中，然而有一种特殊情况：**

* 源进程P1的线程A向目标进程发起了一个请求T1，该请求被分发给目标进程P2的线程B处理
* 源进程P1的线程A等待目标进程P2的线程B处理处理完成请求T1
* 目标进程P2的线程B在处理请求T1的过程中，需要向源进程P1发起另一个请求T2
* 源进程P1除了线程A是处于空闲等待状态之外，还有另外一个线程C处于空闲等待状态

**这时候请求T2应该分发给线程A处理，还是线程C处理？**

* 如果分发给线程C处理，则线程A仍然是处于空闲等待状态
* 如果分发给线程A处理，则线程 C可以处理其它新的请求

**Binder线程的事务堆栈**  
![](https://box.kancloud.cn/9c54725660f70372b4c2d3f953d815b2_612x299.png)  
![](https://box.kancloud.cn/02f5f1f76adca2c74fd579fb5f61efbd_516x316.png)

**T1：From P1 to P2**

* BC\_TRANSACTION:
  * T1-
    &gt;
    from\_parent = NULL
  * T1-
    &gt;
    from = Thread\(A\)
  * Thread\(A\)-
    &gt;
    transaction\_stack = T1
  * Thread\(A\)-
    &gt;
    proc = P1
* BR\_TRANSACTION:
  * T1-
    &gt;
    to\_parent = NULL
  * T1-
    &gt;
    to\_thread = Thread\(B\)
  * Thread\(B\)-
    &gt;
     transaction\_stack = T1

**T2：From P2 to P1**

* BC\_TRANSACTION:
  * T2-
    &gt;
    form\_parent = T1
  * T2-
    &gt;
    from = Thread\(B\)
  * Thread\(B\)-
    &gt;
    transaction\_stack = T2
  * Thread\(B\)-
    &gt;
    proc = P2
* BR\_TRANSACTION
  * T2-
    &gt;
    to\_parent = T1
  * T2-
    &gt;
    to\_thread = Thread\(A\)
  * Thread\(A\)-
    &gt;
    transaction\_stack = T2

**同步请求优先于异步请求**

* 同一时刻，一个BBinder只能处理一个异步请求
* 第一个异步请求将被保存在目标进程的todo list中
* 第一个异步请求未被处理前，其它的异步请求将被保存在对应的 binder\_node的async todo list中
* 第一个异步请求被处理之后，第二个异步请求将从binder\_node的async todo list转移至目标进程的todo list等待处理
* 依次类推……
* 此外，所有同时进行的异步请求所占用的内核缓冲区大小不超过目标进程的总内核缓冲区大小的一半

**与请求相关的三个线程优先级**

* 源线程的优先级
* 目标线程的优先级
* 目标binder\_node的最小优先级（注册Service时设置）

**Binder线程处理同步请求时的优先级**

* 取{源线程，目标binder\_node，目标线程}的最高优先级
* 保证目标线程以不低于源线程优先级或者binder\_node最小优先级的优先级运行

**Binder线程处理异步请求时的优先级**

* 取{目标binder\_node，目标线程}的最高优先级
* 保证目标线程以不低于binder\_node最小优先级的优先级运行

**Binder进程间通信机制的Java接口**

* 每一个Java层的Binder本地对象\(Binder\)在C++层都对应有一个JavaBBinder对象，后者是从C++层的BBinder继承下来的
* 每一个Java层的Binder代理对象\(BinderProxy\)在C++层都对应有一个BpBinder对象
* 于是Java层的Binder进程间通信实际上就是通过C++层的BpBinder和BBinder来进行的，与C++层的Binder进程间通信 一致

#### **Android Ashmem驱动系统**

**传统的Linux共享内存机制**

* System V：shmget、shmctl、shmat、shmdt
  * 用一个整数ID来标志一块共享内存
  * 不能动态释放部分共享内存
* Posix：shm\_open、ftruncate、mmap、munmap
  * 用一个文件描述符来标一块共享内存
  * 不能动态释放部分共享内存
 
    **Android匿名共享内存**
* open、ioctl、mmap、munmap
  * 用一个文件描述符来标一块共享内存
  * 能动态释放部分共享内存

**整体架构**  
![](https://box.kancloud.cn/6ead76b15b5051f0a44b2c619326e7de_666x417.jpg)

**Ashmem驱动程序**

* 与System V的共享内存一样，都是基于临时文件系统\(tmpfs\)来实现的，也就是每一块共享内存都对应有一个临时文件
* 可以通过IO控制命令ASHMEM\_PIN对部分共享内存进行锁定
* 可以通过IO控制命令ASHMEM\_UNPIN对部分共享内存进行解锁
* 没有锁定的部分共享内存，在系统内存紧张时会被回收

**C访问接口**

* ashmem\_create\_region：创建匿名共享内存
  * open /dev/ashmem
  * mmap /dev/ashmem
* ashmem\_pin\_region：锁定部分匿名共享内存
  * ioctl ASHMEM\_PIN
* ashmem\_unpin\_region：解锁部分匿名共享内存
  * ioctl ASHMEM\_UNPIN

**C++访问接口**

* MemoryHeapBase
  * 用来访问一块匿名共享内存
  * Binder Service，实现IMemoryHeap接口
  * 对应的Binder代理为BpMemoryHeap
* MemoryBase： Binder Service
  * 在MemoryHeapBase的基础上实现，用来访问一整块匿名共享内存的其中一部分
  * Binder Service ，实现IMemory接口
  * 对应的Binder代理为BpMemory

**Java访问接口**  
**android.os.MemoryFile**  
![](https://box.kancloud.cn/8714c64a4dc45696617cf0b2599b6acd_728x129.png)  
![](https://box.kancloud.cn/ead93d30736f604e68ae891301d72c23_720x228.png)

**Android 2.3之后，第二个构造函数已不存在，因此MemoryFile只能作为内存文件使用**

**Ashmem进程间共享原理--文件、文件结构体、文件描述符的关系**  
![](https://box.kancloud.cn/4deb683e1219f69b0fcf3ea3ff188fed_731x389.jpg)  
**Ashmem进程间共享原理--两个在不同进程中的文件描述符对应同一个指向设备文件/dev/ashmem的文件结构体**  
![](https://box.kancloud.cn/357558620c7833fd293032bbd7cc0974_512x391.jpg)  
**可以通过Binder IPC在进程间进行传递和共享 – flat\_binder\_object\(BINDER\_TYPE\_FD\)**  
![](https://box.kancloud.cn/9113da10f01d2118d885335782f665b9_601x447.png)

