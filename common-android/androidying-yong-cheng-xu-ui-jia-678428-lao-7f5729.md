### **概述**

Android系统采用一种称为Surface的UI架构为应用程序提供用户界面。在Android应用程序中，每一个Activity组件都关联有一个或者若干个窗口，每一个窗口都对应有一个Surface。有了这个Surface之后，应用程序就可以在上面渲染窗口的UI。最终这些已经绘制好了的Surface都会被统一提交给Surface管理服务SurfaceFlinger进行合成，最后显示在屏幕上面。无论是应用程序，还是SurfaceFlinger，都可以利用GPU等硬件来进行UI渲染，以便获得更流畅的UI。在Android应用程序UI架构中，还有一个重要的服务WindowManagerService，它负责统一管理协调系统中的所有窗口，例如管理窗口的大小、位置、打开和关闭等。

这系列讲Android应用程序的Surface机制，阐述Activity、Window和View的关系，以及应用程序、WindowManagerService和SurfaceFlinger协作完成UI渲染的过程。

* **Android UI架构概述**
* **Android应用程序UI框架**
* **WindowManagerService**
* **SurfaceFlinger**
* **Android多屏支持**

#### **总体架构**

![](https://box.kancloud.cn/dde130147d42ddd25e8ab915a628c9cf_697x608.png)

#### **窗口\(Window\)的结构**

* ViewRootImpl是一个虚拟根View，用来控制窗口的渲染，以及用来与WindowManagerService、SurfaceFlinger通信
* DecorView是窗口的真正根View
* ContentView描述窗口的主题风格
 
  ![](https://box.kancloud.cn/4e2c65511ee02ec7b370665c673f6b47_350x402.png)

**Window与Activity的关系**  
![](https://box.kancloud.cn/65d727d00bc725f1310f47458e8a2fe8_573x433.jpg)

**Activity所对应的Window实际上是一个PhoneWindow**  
![](https://box.kancloud.cn/2f062cf8f154fef7e1bd760add86b477_449x338.jpg)

**Activity/Window的上下文**  
![](https://box.kancloud.cn/e463816eaa963e06c78334b049c07522_612x444.jpg)

**Window的虚拟根View -- ViewRootImpl**  
![](https://box.kancloud.cn/8996c2200c81f2c8439948422e699233_826x472.jpg)

**窗口绘图表面 -- Surface**  
![](https://box.kancloud.cn/014be9494feca7566edd02af6aa7bfa6_463x459.jpg)

**窗口标志 -- W**  
![](https://box.kancloud.cn/a8c24b3ca04ea50f831c5a31e3ee5b33_760x420.jpg)

**窗口会话 -- Session**  
![](https://box.kancloud.cn/35ceb7b71641d8c60b3445b81c5fb2ba_710x300.jpg)

**窗口视图 -- View**  
![](https://box.kancloud.cn/5668b8ddf3b6ee9bdce5b39d4eb0a65d_470x351.jpg)

#### **Android应用程序UI的绘制过程**

![](https://box.kancloud.cn/bee348cd02f4c389909fc7884a6fbedb_519x351.png)

**软件渲染过程**  
![](https://box.kancloud.cn/b7f8412c22c7d87007363090b407db78_416x325.png)

**硬件渲染过程**  
![](https://box.kancloud.cn/dbf1f54ac14a94522ee87a71f30e9b67_538x591.png)

**Display List是什么？**  
Display List是一个缓存绘制命令的Buffer  
**Display List的好处？**  
当View的某些属性发生改变时，只需要修改相应的Buffer中对应的属即可，例如Alpha属性，而无需对整个View进行重绘

#### **Android应用程序UI的绘制时机 – Without Vsync -- Jank**

![](https://box.kancloud.cn/9143f0807b9ec8eea379610a8ef429b4_692x262.png)

#### **Android应用程序UI的绘制时机 – With VSync**

![](https://box.kancloud.cn/496e3c777d83a355b3cb537f5449e646_678x193.png)

#### **Android应用程序UI的绘制时机 – With Vsync and Double Buffering**

![](https://box.kancloud.cn/0e1c48eab72ccd818c2767b280d80877_667x189.png)

#### **Android应用程序UI的绘制时机 – With Vsync and Triple Buffering**

![](https://box.kancloud.cn/5d60a3b92acd8a8d2f0288555261a122_668x263.png)

#### **Android系统的VSync实现**

* SurfaceFlinger内部维护有一个EventThread，用来监控显卡的VSync事件
* Android应用程序通过注册一个DisplayEventReceiver来接收SurfaceFlinger的VSync事件
* Android应用程序接收到重绘UI请求，通过前面注册的DisplayEventReceiver向SurfaceFlinger请求在下一个VSync事件到来时产生一个VSync通知
* Android应用程序获得VSync通知的时候，才会真正执行重绘UI的请求

#### **WindowManagerService**

**职责**

* 计算窗口大小
* 计算窗口Z轴位置
* 管理输入法窗口
* 管理壁纸窗口
* 执行窗口切换

**屏幕的基本结构**  
![](https://box.kancloud.cn/01da93597c4dce354113540b0f261997_651x439.jpg)

**计算窗口大小 – Content Region**  
![](https://box.kancloud.cn/696afa988f7c6323ea2973eb5e03bd66_918x424.jpg)

**计算窗口大小 – Visible Region**  
![](https://box.kancloud.cn/d22f4fb7a0c2f667f0085eafb2fbc610_916x448.jpg)

**计算窗口Z轴位置 – Window Stack**  
![](https://box.kancloud.cn/00adaab7deaa12dcebf865015ecb3e9f_750x524.jpg)

\*\*计算窗口Z轴位置 – 计算时机 \*\*  
![](https://box.kancloud.cn/fa3484bdd7ea0416ffe24220291eae97_817x352.jpg)

**计算窗口Z轴位置 – 计算公式**

**Z = Base Layer + WINDOW\_LAYER\_MULTIPLIER\(5\)  
Base Layer = T \* TYPE\_LAYER\_MULTIPLIER\(10000\) + TYPE\_LAYER\_OFFSET\(1000\)**

**计算窗口Z轴位置 – 窗口主类型**  
![](https://box.kancloud.cn/8bfa0e12277a79b4856b22d7fdaddd53_580x482.png)

**计算窗口Z轴位置 – 窗口子类型**  
![](https://box.kancloud.cn/5db2c6cceed2d96e0b16244a6a2345d9_767x381.jpg)

**管理输入法窗口**  
![](https://box.kancloud.cn/2ab9d3d4e973a65188da5c8524fadf42_573x351.jpg)

**输入法窗口在Window Stack的位置**  
![](https://box.kancloud.cn/01921204c061b8a801ad7fd55abc5fc6_747x426.jpg)

**管理壁纸窗口**  
![](https://box.kancloud.cn/3f0cfd12e5eb36ea4bf822afd0f9b1e2_611x278.jpg)

**壁纸窗口在Window Stack的位置**  
![](https://box.kancloud.cn/4d39e694cfed386a2abbb9ee2568d39e_746x429.jpg)

**执行窗口切换**  
![](https://box.kancloud.cn/938df5cb8d41fb89ec1dcca903c44f3f_524x283.jpg)

**执行窗口切换 – Starting Window**  
![](https://box.kancloud.cn/1fe9bbf69f7561314ade7fc7eac5f5e3_474x561.jpg)

**执行窗口切换 – 动画**  
![](https://box.kancloud.cn/3c0e92d99924024e59017bb0d715cdaf_726x373.jpg)

#### **SurfaceFlinger**

**职责**

* 分配图形缓冲区
* 合成图形缓冲区
* 管理VSync事件

**渲染流程**  
![](https://box.kancloud.cn/31e07a627c009d7bbafe19dede1d2d12_628x500.jpg)  
**分配图形缓冲区**  
![](https://box.kancloud.cn/0f8a88fd5c61278339445e511757d1f1_749x420.png)  
**合成图形缓冲区**  
![](https://box.kancloud.cn/a1aad0e020e1122159e057eba62bd547_747x517.png)  
**HWComposer实例：高通MDP4.0**  
![](https://box.kancloud.cn/7efd956f732fb3f86b6fb356b6cfe877_635x571.jpg)  
**合成图形缓冲区 – 可见性计算**  
![](https://box.kancloud.cn/9eb6c60aae53f049d94ee3950079c5a9_495x253.jpg)

**管理VSync事件**  
![](https://box.kancloud.cn/033f4b7f179af33cc37c264f29fa322e_764x353.png)

#### **Android多屏支持**

**从4.2开始支持多屏幕**  
![](https://box.kancloud.cn/a19f30b95c3746654f0992be1bf1d043_564x355.png)  
**屏幕类型**

* Primary Display

  * 设备自带的屏幕
  * 由SurfaceFlinger管理

* External Display

  * 通过HDMI连接
  * 由SurfaceFlinger监控和管理

* Virtual Display

  * 通过Miracast连接\(基于Wifi Direct技术\)
  * 由DisplayManagerService监控和管理

* App通过android.app.Presentation接口在指定的屏幕上创建窗口  
  [http://developer.android.com/reference/android/app/Presentation.html](http://developer.android.com/reference/android/app/Presentation.html)  
  ![](https://box.kancloud.cn/c0ae5ad20d4565c3b78b642dfd99a4b4_755x198.png)  
  ![](https://box.kancloud.cn/d04836328d3c8d2c952fa4c44b266290_521x59.png)



