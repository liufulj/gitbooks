# **Android系统架构概述**

Android系统 = Linux内核 + Android运行时。

Android系统使用的Linux内核包含了一些专用驱动，例如Logger、Binder、Ashmem、Wakelock、Low-Memory Killer和Alarm等，这些Android专用驱动构成了Android运行时的基石。Android运行时从下到上又包括了HAL层、应用程序框架层和应用程序层。HAL层主要是为规避GPL而设计的，它将将硬件驱动分成内核空间和用户空间两部分，其中用户空间两部分采用的是商业友好的Apache License。应用程序框架层主要包括系统服务，例如组件管理服务、应用程序安装服务、窗口管理服务、多媒体服务和电信服务等。应用程序框架进一步又分为C/C++和Java两个层次，Java代码运行Dalvik虚拟机之上，并且通过JNI方法和C/C++交互。应用程序层主要就是由四大组件Activity、Service、Broadcast Receiver和Content Provider构成，它们是应用开发的基础。

从一个通用的应用程序架构开始，概述Android系统的专用驱动、HAL、关键服务、Dalvik、窗口机制和四大组件等。帮助进一步了解Android系统的具体实现。

* ## Android系统整体架构

![](https://box.kancloud.cn/a17112a1a557ee7c5150884b7a221d0c_857x91.png)

![](https://box.kancloud.cn/23a4b102224f5e01287b381f2ff95d49_730x524.jpg)

* ## Android专用驱动

#### **Logger**

* 完全内存操作
* 适合频繁读写

  ![](https://box.kancloud.cn/d1ad086e660cc704200cfa8966f6c4b0_573x507.jpg)

#### **Binder**

* Client/Server模型
* 进程间一次数据拷贝
* 进程内直接调用

  ![](https://box.kancloud.cn/887eded306fad9269391aa8da0a43c95_784x465.jpg)

#### **Ashmem**

* 使用文件描述符描述
* 通过Binder在进程间传递

  ![](https://box.kancloud.cn/461a77ced95a2b523a5249893ec39bc8_665x418.jpg)

* ## Android硬件抽象层HAL

#### **设备驱动分为内核空间和用户空间两部分**

* 保护厂商利益（出发点）
* 内核空间主要负责硬件访问逻辑（GPL）
* 用户空间主要负责参数和访问流程控制（Apache License）

#### **用户空间部分设备驱动即为HAL Module**

* HAL Module通过设备文件访问内核空间部分设备驱动

#### **系统服务通过HAL Module对硬件进行管理**

* 系统服务通过JNI访问HAL Module

#### **应用程序通过系统服务对硬件进行访问**

* 应用程序通过Binder IPC访问系统服务

#### **整体架构图**

![](https://box.kancloud.cn/993b4084be40134e34456e2907347f30_722x454.jpg)

* ## Android应用程序组件

#### **Android应用程序的一般架构**

![](https://box.kancloud.cn/403b3a15a93f52469a256983835adf57_814x518.jpg)

#### **Android应用程序的一般架构**

* Activity -- UI、交互
* Service -- 后台计算
* Broadcast Receiver -- 广播
* Content Provider -- 数据

![](https://box.kancloud.cn/229a00062b0dd89ef23b9f3ccd6180b4_925x462.png)

#### **Activity生命周期**

由ActivityManagerService管理  
![](https://box.kancloud.cn/5f620cd5ed7ba1f5d5224f05dbc0ec65_451x575.jpg)

#### **Activity堆栈**

由ActivityManagerService维护  
![](https://box.kancloud.cn/dc3e75f0c05c893cc641be28bed9926c_619x201.jpg)

#### **Activity在堆栈中以Task的形式聚集在一起**

**Task由一系列相关的Activity组成，描述用户完成某一个操作所需要的Activity  
当我们从Launcher上点击一个应用图标的时候，就启动一个Task  
Task是用Android多任务的一种体现      
**[**http://developer.android.com/guide/components/tasks-and-back-stack.html**](http://developer.android.com/guide/components/tasks-and-back-stack.html)  
![](https://box.kancloud.cn/111d6baea5440ff27d19e8ab639d8e9c_300x171.jpg)

#### **Service**

* Unbounded service
* Bounded service

  ![](https://box.kancloud.cn/d9cc669283608a4df2c5b0bdc595c83c_406x512.jpg)

#### **Broadcast Receiver**

* 注册
  * 静态 -- AndroidManifest.xml
  * 动态 -- Context.registerReceiver
* 广播
  * 无序 -- Context.sendBroadcast
  * 有序 -- Context.sendOrderedBroadcast
* **注册广播**

  ![](https://box.kancloud.cn/e3ca3b92735f0d6505aec285aee1f98b_651x268.png)

* **发送广播**

  ![](https://box.kancloud.cn/a02a367c95193289e3262e37ffcc7ded_986x335.png)

#### **Content Provider**

* 通过URI来描述

* 数据访问接口

* 数据更新机制

* **Content Provider的URI结构**

  * A -- Scheme
  * B -- Authority
  * C -- Resource Path
  * D -- Resource ID

    ![](https://box.kancloud.cn/5ae216e8926e46d551fab5462013e805_537x84.png)

* **Content Provider数据访问接口**

  * Insert
  * Update
  * Delete
  * Query
  * Call -- Hidden

* **Content Provider数据更新机制**

  * 注册内容观察者 -- ContentResolver.ContentObserver
  * 发送数据更新通知 -- ContentResolver.notifyChange

* **注册Content Provider的内容观察者**  
  ![](https://box.kancloud.cn/056b69744fe6a7c5cd69efca73f79b28_589x252.png)

* **发送Content Provider数据更新通知**  
  ![](https://box.kancloud.cn/63cea9d69df6705e75716fdbd9856693_802x247.png)

* ## Android应用程序框架
* 管理硬件

* 提供服务

* 组件管理

* 进程管理

#### **按服务类型划分**

* **Hardware Service**
  * CameraService
  * LocationManagerService
  * LightsService
  * ……
* **Software Service**
  * PackageManagerService
  * ActivityManagerService
  * WindowManagerService
  * ……

#### **按开发语言划分**

* Java Runtime Framework
  * PackageManagerService
  * ActivityManagerService
  * WindowManagerService
  * ……
* Native Runtime Framework
  * MediaPlayerService
  * SurfaceFlinger
  * AudioFlinger
  * ……

#### **按进程划分**

* System Server Process
  * PackageManagerService
  * ActivityManagerService
  * WindowManagerService
  * ......
* Independent Process
  * SurfaceFlinger
  * MediaPlayerService
  * ……

#### **服务注册、获取和访问过程**

![](https://box.kancloud.cn/f860ed717a83940e80b7c23908a6cb41_746x290.png)

* ## Android用户界面架构

#### **窗口管理框架**

* Window
* WindowManagerService
* SurfaceFlinger
 
  ![](https://box.kancloud.cn/8bc177c30dbdbd85c96adf9cba796e90_698x608.png)

#### **资源管理框架**

* AssetManager
* Resources
 
  ![](https://box.kancloud.cn/52ade3f62b423568fa389cae5fdf64f5_580x559.jpg)

* ## Dalvik虚拟机

#### **Java虚拟机与Dalvik虚拟机区别**

![](https://box.kancloud.cn/56881e3bb303f1470ff3ccbb7ff3f01d_1152x356.png)

#### **Dex文件编译和优化**

![](https://box.kancloud.cn/ca6ae5914e857eeaf3384f4b7a13b858_362x224.png)

#### **内存管理**

* Java Object Heap
 
  大小受限，16M/24M/32M/48M
* Bitmap Memory\(External Memroy\)：
 
  大小计入Java Object Heap
* Native Heap
 
  大小不受限

#### **垃圾收集\(GC\)**

* Mark，使用RootSet标记对象引用
* Sweep，回收没有被引用的对象

#### **GingerBread之前**

* Stop-the-word，也就是垃圾收集线程在执行的时候，其它的线程都停止
* Full heap collection，也就是一次收集完全部的垃圾
* 一次垃圾收集造成的程序中止时间通常都大于100ms

#### **GingerBread之后**

* Cocurrent，也就是大多数情况下，垃圾收集线程与其它线程是并发执行的
* Partial collection，也就是一次可能只收集一部分垃圾
* 一次垃圾收集造成的程序中止时间通常都小于5ms

#### **即时编译\(JIT\)**

* 从2.2开始支持JIT，并且是可选的，编译时通过WITH\_JIT宏进行控制
* 基于执行路径\(Executing Path\)对热门的代码片断进行优化\(Trace JIT\)，传统的Java虚拟机以Method为单位进行优化\(Method JIT\)
* 可以利用运行时信息进行激进优化，获得比静态编译语言更高的性能
* 实现原理：
  [http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html](http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html)

#### **Java本地调用\(JNI\)**

* 实现Java与C/C++代码互调
* 大部分Java接口的都是通过JNI调用C/C++接口实现的
* 提供有NDK进行JNI开发

#### **进程和线程管理**

* 与Linux进程和线程一一对应
* 通过fork系统调用创建进程
* 通过pthread库接口创建线程



