### **概述**

Android系统里面的应用程序进程有一个特点，那就是它们是被系统托管的。也就是说，系统根据需要来创建进程以及回收进程。进程创建发生在组件启动时，它们是由Zygote进程负责创建。Zygote进程是由系统中的第一个进程init负责启动。此外，用来运行各种系统服务的System Server进程也是由Zygote进程创建的。进程回收发生在内存紧张时，由Low Memory Killer执行。此外，组件管理服务ActivityManagerService和窗口管理服务WindowManagerService也会在适当的时候主动进行进程回收。每一个应用程序进程根据运行情况被赋予优先级，当需要回收进程的时候，就按照优先级从低到高的顺序进行回收。  
主要讲Android应用程序进程的启动和回收，主要涉及到Zygote进程、System Server进程，以及组件管理服务ActivityManagerService、窗口服务WindowManagerService，还有专用驱动Low Memory Killer。通过了解Android系统对应用程序进程的管理，我们就能更清楚应用程序的运行机制。

* Android系统启动概述
* Zygote进程启动过程分析
* System Server进程启动过程分析
* Android应用程序进程启动过程分析
* Android应用程序进程回收机制

#### **Android系统启动概述**

![](https://box.kancloud.cn/645e9c852b0f6a8589b988fa74d363a4_654x574.jpg)

#### **Zygote进程启动过程分析**

**Zygote进程由Init进程启动**  
![](https://box.kancloud.cn/09c3c00db3db5d603e73be7b9c703a54_645x142.png)

* 加载文件：/system/app\_process
* --start-system-server：启动System Server进程
* 创建名称为zygote的socket：用来和ActivityManagerService通信

**app\_process**  
![](https://box.kancloud.cn/8d48058d23772392e460f8030809b43e_519x304.png)

**AndroidRuntime::start**  
![](https://box.kancloud.cn/6ee694ef0f6eddd177a6ea921f94ff19_599x511.png)

**启动Dalvik虚拟机**

* 创建一个Dalvik虚拟机实例
* 加载Java核心类及其JNI方法
* 初始化主线程的JNI环境

**加载部分Android核心类及其JNI方法**

* android.os.\*
* android.graphics.\*
* android.opengl.\*
* android.hardware.\*
* android.media.\*
* ……

**ZygoteInit.main**  
![](https://box.kancloud.cn/ec79009573d4dad2ee87a99bbbc074f2_455x433.png)

**Preload Classes**

* 参考frameworks/base/preloaded-classes文件
  * android.accounts.\*
  * android.app.\*
  * android.view.\*
  * ……

![](https://box.kancloud.cn/0546d509c4350bafba5af7450d44ce73_645x358.png)

**Preload Drawables**

* 参考frameworks/base/core/res/res/values$/arrays.xml文件
  * @drawable/toast\_frame\_holo
  * @drawable/btn\_check\_on\_pressed\_holo\_light
  * @drawable/btn\_check\_on\_pressed\_holo\_dark
  * ……

![](https://box.kancloud.cn/ccc4077543342863d109796d135de604_644x362.png)

**Preload Color State List**

* 参考frameworks/base/core/res/res/values$/arrays.xml文件
  * @color/primary\_text\_dark
  * @color/primary\_text\_dark\_disable\_only
  * @color/primary\_text\_dark\_nodisable
  * ……
 
    ![](https://box.kancloud.cn/07dbe665b359697928bee34030265864_644x351.png)

**runSelectLoopMode**  
![](https://box.kancloud.cn/57777c0cba975060a3c84b2aeef345e5_556x561.png)

**Zygote进程启动完成后的地址空间**  
![](https://box.kancloud.cn/80e15f559f49997ed9ae57eff6dace18_591x447.png)

#### **System Server进程启动过程分析**

**Zygote在启动的过程中创建System Server进程**  
![](https://box.kancloud.cn/41fe6a64c19d5dec909ddd237b73ee1d_455x433.png)  
**startSystemServer**  
![](https://box.kancloud.cn/b2bfa15b915443cd2720296023330750_753x564.png)

**备注**  
用户组ID定义参考system/core/include/private/android\_filesystem\_config.h  
Capability权限定义参考kernel/goldfish/include/linux/capability.h

**handleSystemServerProcess**  
![](https://box.kancloud.cn/4a2a8bf4306efb417f7370bb74bc88f8_884x431.png)

**RuntimeInit.zygoteInit**  
![](https://box.kancloud.cn/0df5b6e1c59f6bdf3b6baab6baf9c590_695x185.png)

**nativeZygoteInit--启动Binder线程池**  
![](https://box.kancloud.cn/76b1a1d777c548e15310ef7a17bbff36_841x70.png)  
![](https://box.kancloud.cn/f271357457d502cd24efb9e5435eb924_512x242.png)

**applicationInit—调用SystemServer.main**  
![](https://box.kancloud.cn/7c820776e1f920ebc3890235c7190541_703x358.png)

**SystemServer.main**  
![](https://box.kancloud.cn/50a258cfc64516c90c30668b859210a3_452x190.png)

**Init1—启动C/C++ Rutime Framework Service**  
![](https://box.kancloud.cn/5584d00313d6410a1787d005df26cc72_556x564.png)

**Init2—启动Java Runtime Framework Service**  
![](https://box.kancloud.cn/40f79adb2fcad60ae59887192011c571_490x198.png)

**ServerThread.run**  
![](https://box.kancloud.cn/c94e0ec28b620f8f99264e0303092959_614x533.png)

**System Server进程启动完成后的地址空间**  
![](https://box.kancloud.cn/3508c7c8e1fe9fb500b6d2e1d5d949f2_675x465.png)

#### **Android应用程序进程启动过程分析**

**ActivityManagerService.startProcessLocked**  
![](https://box.kancloud.cn/a92b45a7de116e86a1e1799f0f44a9a8_695x556.png)

**Process.start**  
![](https://box.kancloud.cn/66e4361b6275323d946789054fe9f383_690x338.png)

**Process.startViaZygote**  
![](https://box.kancloud.cn/b4bf6ed73eb846dd10a07ec89d3691e7_652x558.png)

**Process.zygoteSendArgsAndGetResult**  
![](https://box.kancloud.cn/97d2c2b39ed1bf96db82fc7b684f70ac_629x527.png)

**ZygoteConnection.runOnce**  
![](https://box.kancloud.cn/1bc4a18848ad3401cfa82e9c58c1e67f_682x559.png)

**ZygoteConnection.handleChildProc**  
![](https://box.kancloud.cn/1b2c78950a039f17dddb6618f17d530a_794x309.png)

**RuntimeInit.zygoteInit**

* nativeZygoteInit
* applicationInit
  * Invoke main of ActivityThread

**ActivityThread.main**  
![](https://box.kancloud.cn/754b52ab968f4ebb332f41d3b9889497_684x273.png)

#### **Android应用程序进程回收机制**

**Linux的内存回收机制--Out of Memory Killer**

* 每一个进程都有一个oom\_adj值，取值范围\[-17,15\]，可以通过
  `/proc/`
  `<`
  `pid`
  `>`
  `/oom_a`
  dj访问
* 每一个进程的oom\_adj初始值都等于其父进程的oom\_adj值
* oom\_adj值越小，越不容易被杀死，其中，-17表示不会被杀死
* 内存紧张时，OOM Killer综合进程的内存消耗量、CPU时间、存活时间和oom\_adj值来决定是否要杀死一个进程来回收内存

**备注**：  
oom\_adj值定义可以参考kernel/goldfish/include/linux/oom.h

**Android的内存回收机制—Low Memory Killer**

* 进程的oom\_adj值由ActivityManagerService根据运行在进程里面的组件的状态来计算
* 进程的oom\_adj值取值范围为\[-16,15\]， oom\_adj值越小，就不容易被杀死
* 内存紧张时， LMK基于oom\_adj值来决定是否要回收一个进程
* ActivityManagerService和WindowManagerService在特定情况下也会进行进程回收

**LMK的进程回收策略**

* 当系统内存小于i时，在oom\_adj值大于等于j的进程中，选择一个oom\_adj值最大并且消耗内存最多的进程来回收  
  ![](https://box.kancloud.cn/2d71992bf1b34112c5419c2c4b8326ca_685x366.png)

* 应用程序进程的oom\_adj值

  * SYSTEM\_ADJ\(-16\)：System Server进程
  * PERSISTENT\_PROC\_ADJ\(-12\)：android:persistent属性为true的系统App进程，如PhoneApp
  * FOREGROUND\_APP\_ADJ\(0\)：包含前台Activity的进程
  * VISIBLE\_APP\_ADJ\(1\)：包含可见Activity的进程
  * PERCEPTIBLE\_APP\_ADJ\(2\)：包含状态为Pausing、Paused、Stopping的Activity的进程，以及运行有Foreground Service的进程
  * HEAVY\_WEIGHT\_APP\_ADJ\(3\)：重量级进程， android: cantSaveState属性为true的进程，目前还不开放
  * BACKUP\_APP\_ADJ\(4\)：正在执行备份操作的进程
  * SERVICE\_ADJ\(5\)：最近有活动的Service进程
  * HOME\_APP\_ADJ\(6\)：HomeApp进程
  * PREVIOUS\_APP\_ADJ\(7\)：前一个App运行在的进程
  * SERVICE\_B\_ADJ\(8\)：SERVICE\_ADJ进程数量达到一定值时，最近最不活动的Service进程
  * HIDDEN\_APP\_MIN\_ADJ\(9\)和HIDDEN\_APP\_MAX\_ADJ\(15\)：含有不可见Activity的进程，根据LRU原则赋予\[9,15\]中的一个值

* Init进程的oom\_adj值被设置为-16，由Init进程所启动的daemon和service进程的oom\_adj值也等于-16

* 如果运行在进程A中的Content Provider或者Service被绑定到进程B，并且进程B的oom\_adj值比进程A的oom\_adj小，那么进程A的oom\_adj值就会被设置为进程B的oom\_adj值，但是不能小于FOREGROUND\_APP\_ADJ

**注意**：  
android: cantSaveState保存在ApplicationInfo中，ApplicationInfo由PackageManagerService维护，应用程序不能修改

**ActivityManagerService在以下四种情况下会更新应用程序进程的oom\_adj值，以及杀掉那些已经被卸载了的App所运行在的应用程序进程**

* activityStopped：停止Activity
* setProcessLimit：设置进程数量限制
* unregisterReceiver：注销Broadcast Receiver
* finishReceiver：结束Broadcast Receiver
 
  **WindowManagerService在处理窗口的过程中发生Out Of Memroy时，也会通知ActivityManagerService杀掉那些包含有窗口的应用程序进程**



