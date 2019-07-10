# Android组件设计思想

### **概述**

Android应用开发的哲学是把一切都看作是组件。把应用程序组件化的好处是降低模块间的耦合性，同时提高模块的复用性。Android的组件设计思想与传统的组件设计思想最大的区别在于，前者不依赖于进程。也就是说，进程即使由于内存紧张被强行杀掉了，但是运行在里面的组件还是存在的。这样就可以在组件再次需要使用时，原地满血复活，就像什么都没发生过一样。这种设计思想非常适合内存较小的移动设备。  
理解Android组件设计思想，对Android应用程序架构会有更好的认识。这一节讲Android组件化设计的背景、理念、原则，以及Android在OS级别上提供的组件化支持，其中还会包含一个实验来验证这种组件化设计思想，可以对Android系统有一个高层次的抽象理解。

* 组件化背景
* 组件化设计
* 组件化支持
* 一个小实验

### **组件化背景**

#### **从PC客户端应用程序说起**

* 开发者角度
  * 复杂，同时兼顾UI、交互和业务逻辑
  * 运行载体是进程
  * 进程只有一个入口点—main
* 使用者角度
  * 流畅的UI、友好的交互、正确的结果
  * 不知进程是何物

#### **PC客户端应用程序开发的任务**

* 满足用户的需求
* 降低程序复杂度 -- 组件化

#### PC客户端应用程序组件化之后

* 开发者角度
  * 运行载体仍然是进程
  * 进程仍然是只有一个入口点—main
* 使用者角度
  * 流畅的UI、友好的交互、正确的结果
  * 不知进程是何物

#### **结论**

应用程序组件化前后，用户对其运行载体（进程）没有概念，但是开发者来说，组件化后的应用程序仍然是和进程直接关联的，也就是说，进程一旦不存在，程序和组件也随之灰飞烟灭！

#### **再说移动客户端应用程序**

* 与PC客户端应用程序一样，包含UI、交互和业务等复杂逻辑
* 与PC客户端应用程序一样，用户的需求是流畅的UI、友好的交互、正确的结果
* 运行在低频率CPU、小容量内存、小面积屏幕设备上

#### **设备特性对移动客户端应用程序的影响**

* 低频率CPU
 
  影响程序运行速度，尤其是程序启动速度，因为用户对程序启动时间最为敏感
* 小容量内存
 
  影响同时运行的程序的数量，而且系统会在内存紧张时杀进程，以便回收内存
* 小面积屏幕
 
  单窗口操作模式，导致用户需要频繁地切换程序或者重新打开程序

#### **移动客户端应用程序开发的任务**

* 满足用户的需求
* 降低程序复杂度

#### **组件化是降低移动端应用程序复杂度的不二选择，但需进一步考虑以下两个事实**：

* 设备特性的影响
  * 系统杀进程时，被杀进程里面的组件如何处理？
  * 程序切换和重新打开时，如可提高程序启动速度？
* 用户不关心进程
  * 是否可以将组件与进程进行剥离？

### **组件化设计**

#### **基本思想**

Everything is component

#### **具体实现**

* 程序由组件组成
* 组件与进程剥离
* 组件皆程序入口

#### **程序由组件组成**

* Activity：前台交互
* Service：后台计算
* Broadcast Receiver：广播通信
* Content Provider：数据封装

此四种组件对客户端应用程序进行了很好的抽象，QT的signal-slot，COM的连接点，都是广播的例子

#### **组件与进程剥离**

* 组件关闭时，进程可以继续存在
 
  提高重新启动时的速度
* 进程关闭时，组件可以继续存在
 
  保护被杀进程里面的组件

#### **组件皆程序入口**

* Activity -- onCreate
* Service -- onCreate
* Broadcast Receiver -- onReceive
* Content Provider -- onCreate

由于组件已与进程剥离，因此不再有进程入口的概念，只有组件入口的概念

#### **将组件与进程进行剥离，使得进程对组件透明，听起来很好，但是如何解决以下四个问题？**

* 谁来负责组件的启动和关闭？
* 谁来维护组件的状态？
* 谁来管理组件运行时所需要的进程？
* 组件之间如何进行通信？

### **组件化支持**

#### **操作系统级别的组件化支持**

* Activity Manager Service
* Binder
* Low Memory Killer

#### **Activity Manager Service**

* 启动组件

       组件启动时，检查其所要运行在的进程是否已创建。如果已经创建，就直接通知它加载组件。否则，先将该进程创建起来，再通     知它加载组件。

* 关闭组件

        组件关闭时，其所运行在的进程无需关闭，这样就可以让组件重新打开时得到快速启动。

* 维护组件状态
  维护组件在运行过程的状态，这样组件就可以在其所运行在的进程被回收的情况下仍然继续生存。

* 进程管理
  在适当的时候主动回收空进程和后台进程，以及通知进程自己进行内存回收

1. 组件的UID和Process Name唯一决定了其所要运行在的进程。
2. 每次组件onStop时，都会将自己的状态传递给AMS维护。
3. AMS在以下四种情况下会调用trimApplications来主动回收进程：
   A. activityStopped
   B. setProcessLimit
   C. unregisterReceiver
   D. finishReceiver
4. WMS也会主动回收进程：
   WindowManagerService在处理窗口的过程中发生Out Of Memroy时，会调用reclaimSomeSurfaceMemoryLocked来回收某些Surface占用的内存，reclaimSomeSurfaceMemoryLocked的逻辑如下所示：
   \(1\).首先检查有没有泄漏的Surface，即那些Session已经不存在但是还没有销毁的Surface，以及那些宿主Activity已经不可见但是还没有销毁的Surface。如果存在的话，就将它们销毁即可，不用KillPids。
   \(2\).如果不存在没有泄漏的Surface，那么那些存在Surface的进程都有可能被杀掉，这是通过KillPids来实现的。

**KillPids**：

* \(1\). 找中候选进程的最大oom\_adj值。
* \(2\). 如果找到的最大oom\_adj值恰好位于\[ProcessList.HIDDEN\_APP\_MIN\_ADJ\(9\), ProcessList.HIDDEN\_APP\_MAX\_ADJ\(15\)\]之间，那么就将最大oom\_adj值修改为ProcessList.HIDDEN\_APP\_MIN\_ADJ，表示要将所有后台进程都杀掉。
* \(3\). 如果此时杀进程是不安全的，并且找到的最大oom\_adj值比ProcessList.SERVICE\_ADJ还要小，那么就将将最大oom\_adj值设置为ProcessList.SERVICE\_ADJ，避免重要进程被杀掉。
* \(4\). oom\_adj值大于等于前面找到的最大oom\_adj值的候选进程都将被杀掉。

**trimApplications**:

* \(1\). 杀掉Package已经被Remove了的进程，属于无用进程
* \(2\). 调用updateOomAdjLocked更新所有进程的oom\_adj

**updateOomAdjLocked -- all**：

* \(1\). 根据进程运行状态来更新进程的oom\_adj。更新顺序是从最近使用到最近不使用的。如果一个进程是后台进程，那么按照这个顺序进行更新，就意味着最近越是不使用的后台进程，它获得的oom\_adj值就越大。后台进程\(以及空进程\)的oom\_adj取值范围\[ProcessList.HIDDEN\_APP\_MIN\_ADJ\(9\), ProcessList.HIDDEN\_APP\_MAX\_ADJ\(15\)\]
* \(2\)，如果空进程超过限制，那么最近越是不使用的空进程，就越会被杀掉。
* \(3\). 如果后台进程超过限制，那么最近越是不使用的后台进程，就越会被杀掉。
* \(4\). 通知进程ScheduleTrimMemory。
  updateOomAdjLocked -- single：
  * \(1\). 调用computeOomAdjLocked计算oom\_adj。
  * \(2\). 计算oom\_adj之后，如果ShchedGroup发生改变，并且变成了THREAD\_GROUP\_BG\_NONINTERACTIVE调度组，那么进程就会被杀掉。

**computeOomAdjLocked**：

* \(1\). ProcessList.FOREGROUND\_APP\_ADJ\(0\): 前台进程、有InstrumentionClass的进程、正在接收广播的进程、正在执行Service Callback的进程、正在运行有外部依赖的Content Provider的进程
* \(2\). ProcessList.VISIBLE\_APP\_ADJ\(1\): 有Activity是Visible的进程
* \(3\). ProcessList.PERCEPTIBLE\_APP\_ADJ\(2\)：有Activity是Pausing、Paused或者Stopping状态的进程、有Service被设置为Foreground的进程、被设置为Foreground的进程。
* \(4\). ProcessList.HEAVY\_WEIGHT\_APP\_ADJ\(3\)：被设置为重量级App的进程。
* \(5\). ProcessList.BACKUP\_ADJ\(4\)：正在执行备份任务的进程。
* \(6\). ProcessList.SERVICE\_ADJ\(5\)：最近有活动的Service的进程。
* \(7\). ProcessList.HOME\_APP\_ADJ\(6\)：Home App进程。
* \(8\). ProcessList.PREVIOUS\_APP\_ADJ\(7\)：前一个显示UI的进程。
* \(9\). ProcessList.SERVICE\_B\_ADJ\(8\)：oom\_adj值等于ProcessList.SERVICE\_ADJ的进程超过限制时，最近越是不使用的进程的oom\_adj值就会由ProcessList.SERVICE\_ADJ变成ProcessList.SERVICE\_B\_ADJ。

**PS**：

* \(1\). 如果一个进程运行有绑定至另外一个进程（Client）的Content Provider或者Service，并且client进程的oom\_adj值比该进程的oom\_adj小，那么该进程的oom\_adj值就会被设置为client进程的oom\_adj值，但是不能超过ProcessList.FOREGROUND\_APP\_ADJ。
* \(2\). ProcessList.PERSISTENT\_PROC\_ADJ\(-12\)：被设置为persistent的进程，如PhoneApp，不参与oom\_adj更新计算。
* \(3\). ProcessList.SYSTEM\_ADJ\(-16\)：System进程，不参与oom\_adj更新计算。

#### **Binder**

* 为组件间通信提供支持
  * 进程间
  * 进程内
* 高效的IPC机制
  * 进程间的组件通信时，通信数据只需一次拷贝
  * 进程内的组件通信时，跳过IPC进行直接的通信

传统的IPC，通信数据需要执行两次，一次是从源进程的用户空间拷贝到内核空间，二次是从内核空间拷贝到目标进程的用户空间

### **Low Memory Killer**

* 内存紧张时回收进程
  由于组件与进程是剥离的，因此进程回收不会影响组件的生命周期
* 从低优先级进程开始回收
  * Empty Process
  * Hidden Process
  * Perceptible Process
  * Visible Process
  * Foreground Process
* **备注：**

1. 每一个APP进程都有一个oom\_adj值，该值是根据进程所运行的组件计算出来的，值越小，优先级就越级。
2. Init和System Server进程的oom\_adj等于-16，是最高的，保证不会被杀死。
3. PhoneApp具有persist属性，它的oom\_adj被设置为-12，也能保证不会被杀死。
4. 可以通过`/proc/<pid>oom_adj`文件查看进程的oom\_adj值。
5. 在Linux内核中，子进程的oom\_adj值等于父进程的oom\_adj，也就是说，Android里面的Native进程的oom\_adj值与fork它的进程的oom\_adj值一样。

### **一个小实验**

![](https://box.kancloud.cn/6b0a21741ce83aee9bb14f9e02eaec5b_730x427.jpg)  
![](https://box.kancloud.cn/11d7df7d87dfe78872979ca93ee81c0c_376x535.jpg)  
![](https://box.kancloud.cn/2a6bb88d43b77719b55c64f1a74e2dc8_558x498.jpg)  
![](https://box.kancloud.cn/9312ef0cf2b9ba90789c4702ff961435_648x379.jpg)  
![](https://box.kancloud.cn/c77136efc5ba8eaf15202a9f46692f2b_557x510.jpg)  
![](https://box.kancloud.cn/c965bf242f8563dee2137547d511bf9a_376x535.jpg)  
![](https://box.kancloud.cn/a27568f6388372fe758ad74adf42cfed_561x487.jpg)

#### **参考**

[Google安卓开发培训入门指南](http://developer.android.com/training/index.html)  
[Google安卓开发应用组件](http://developer.android.com/guide/components/index.html)  
[Android 源代码](http://source.android.com/source/index.html)  
[Android 接口和架构](http://source.android.com/devices/index.html)

