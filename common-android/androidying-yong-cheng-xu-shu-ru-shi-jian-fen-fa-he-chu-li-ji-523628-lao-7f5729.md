### **概述**

在Android应用程序中，有一类特殊的消息，是专门负责与用户进行交互的，它们就是触摸屏和键盘等输入事件。触摸屏和键盘事件是统一由系统输入管理器InputManager进行分发的。也就是说，InputManager负责从硬件接收输入事件，然后再将接收到的输入事件分发当前激活的窗口处理。此外，InputManager也能接收模拟的输入事件，用来模拟用户触摸和点击等事件。当前激活的窗口所运行在的线程接收到InputManager分发过来的输入事件之后，会将它们封装成输入消息，然后交给当前获得焦点的控件处理。  
主要讲Android应用程序输入事件的分发和处理过程，主要涉及到输入管理InputManager、输入事件监控线程InputReader、输入事件分发线程InputDispatcher，以及应用程序主线程消息循环。

* Android输入系统概述
* 输入管理器的启动过程
* 输入通道的注册过程
* 输入事件的分发过程
* 软键盘输入事件的分发过程

#### **Android输入系统概述**

**输入管理器框架**  
![](https://box.kancloud.cn/4f6d8b5f2b2dfd443090c6b272e25155_681x571.jpg)

**输入管理器与应用程序通过输入通道交互**  
![](https://box.kancloud.cn/69060a95c6b569044ba833adcff037e7_434x201.png)

**输入通道**  
![](https://box.kancloud.cn/f00c0230d6b40bc5635313bb1d541fdf_539x388.jpg)

> 注：mFd指向的是一个socket

#### **输入管理器的启动过程**

**由System Server创建和启动**  
![](https://box.kancloud.cn/0661dfeca45764716f4d05360862b7f0_662x548.png)

注：inputManager.setWindowManagerCallbacks：设置回调，例如用来截获输入事件

**创建InputManagerService**  
![](https://box.kancloud.cn/72e71fba156010b1c123e0b86820faf0_639x263.png)

**创建NativeInputManager**  
![](https://box.kancloud.cn/96fca6b31145c51cab0cddce78312892_894x296.png)

**创建InputManager、InputReader和InputDispatcher，以及InputReaderThread、 InputDispatcherThread**  
![](https://box.kancloud.cn/a42a83147dc6c29edd20ad370b992d9d_633x223.png)

**启动InputManagerService**  
![](https://box.kancloud.cn/72e71fba156010b1c123e0b86820faf0_639x263.png)

**启动NativeInputManager**  
![](https://box.kancloud.cn/96fca6b31145c51cab0cddce78312892_894x296.png)

**启动InputManager**  
![](https://box.kancloud.cn/8a782db31d2e53271d26fefce55534ca_808x293.png)

**启动InputDispatcher**  
**InputDispatcher.dispatchOnce**  
![](https://box.kancloud.cn/52141b141aa8aeb871de3ade1fe714a6_769x410.png)

> 注：InputDispatcher负责分发IO事件，IO事件分为两种类型。一种普通的输入事件，另一种是输入设备本身的事件，例如，输入设备配置发生改变事件。普通的输入事件由dispatchOnceInnerLocked处理，输入设备事件由runCommandsLockedInterruptible处理。InputDispatcher本身也维护着两个队列，一个是mInboundQueue，保存的是普通的输入事件，另一个是mCommandQueue，保存的是输入设备事件。

**启动InputReader**

* **InputReaderThread.threadLoop**  
  ![](https://box.kancloud.cn/b3b603d72f8d5a63a4962c2bcfdf726f_362x78.png)

* **InputReader.loopOnce**  
  ![](https://box.kancloud.cn/c195f5c187f1d43f2c99f850b5ec0d12_797x561.png)

> 注：mNextTimeout：振动设备是按照一定的频率来进行的，这个频率就可以通过设置mNextTimeout来获得

* **EventHub.getEvents**  
  ![](https://box.kancloud.cn/d1e337233c2cf581f1f45da94da44654_898x576.png)

* **EventHub.scanDevicesLocked**  
  ![](https://box.kancloud.cn/49d976fcb4a4cad0c089858daa128884_521x201.png)

> 注：虚拟键盘：设备不一定有键盘，但是程序在测试的时候时候需要模拟键盘输入，这时候模拟的键盘输入就看作是从虚拟键盘发出的。EventHub只存一定会存在虚拟键盘。

* **EventHub.scanDirLocked**
 
  ![](https://box.kancloud.cn/1f32e67f23bd6505f59b581be8d391ae_561x391.png)

> 注：可以用getevent –i命令来查看/dev/input目录下各个文件所对应的输入设备信息

* **EventHub.openDeviceLocked**
 
  ![](https://box.kancloud.cn/3d8fb906ee897c6967794d1c60412a4b_717x576.png)

> 注：Key Map Configuraton File和Configuration File：保存在/system/usr或者/data/system/devices目录下

#### **输入通道的注册过程**

**Activity窗口的组成**  
![](https://box.kancloud.cn/07b0ecdff8c9064793b0f57e91fee43e_581x281.jpg)

> 注：InputChannel是在应用程序请求WMS创建一个Activity窗口时创建的，也就是调用ViewRootImpl.setView开始创建的，在这个过程中，也会同时注册InputChannel，包括Server端InputChannel和客户端InputChannel

\*\*创建InputChannel \*\*

* **ViewRootImpl.setView**
 
  ![](https://box.kancloud.cn/f4f7a176405a74da9209eace18175465_744x482.png)

> 注：mInputQueueCallback不等于null表示由view自己来接管输入事件，否则的话就由ViewRootImpl来接管

* **Session.addToDisplay**  
  ![](https://box.kancloud.cn/8e5f8ae9a10d7e72542638bf23a2de93_796x212.png)

* **WMS.addWindow**  
  ![](https://box.kancloud.cn/4fa0a2c33d54b05e0ed7d0d80d7d9eba_775x575.png)

* **InputChannel.openInputChannelPair**  
  ![](https://box.kancloud.cn/f249c8e21305b8b459d7d4d0b53e627a_593x261.png)

* **nativeOpenInputChannelPair**  
  ![](https://box.kancloud.cn/9c6b18d1a115f0eaad45f761c88ee6df_752x308.png)

* **InputChannel::openInputChannelPair**  
  ![](https://box.kancloud.cn/6c9dd9017ea5c7c46024e5e37f5a14dd_675x432.png)

**注册Server端InputChannel**  
![](https://box.kancloud.cn/8dad7a2078e6ae737b4a2eb4832315d4_653x448.png)

> 注：win.mInputWindowHandle：窗口句柄，用来描述窗口的大小和位置等信息

* **IMS.registerInputChannel**  
  ![](https://box.kancloud.cn/0da664a739be6463d7dd1f5139c54636_671x243.png)

* **nativeRegisterInputChannel**  
  ![](https://box.kancloud.cn/764f59aebd9aa15405512c149b9a20ad_749x244.png)

* **NativeInputManager.registerInputChannel**  
  ![](https://box.kancloud.cn/b510b114dd5f02fa201931f3f3acb4e0_606x103.png)

* **InputDispatcher.registerInputChannel**  
  ![](https://box.kancloud.cn/a447aa5db2fa7ff2e59e6b62b9fcc9ca_759x464.png)

* **Looper.addFd**  
  ![](https://box.kancloud.cn/ce5f24ad89d000d80cffbda106827171_783x51.png)  
  ![](https://box.kancloud.cn/0b1261c6e83aca4587bbab2cb66e78f9_805x539.png)

**更新当前激活窗口**  
![](https://box.kancloud.cn/5aa13d334d3ca065dddff724aa8cc923_659x447.png)

* **WMS.updateFoucsedWindowLocked**
 
  ![](https://box.kancloud.cn/7b9d0fc1bc35840c783c026c3cc84000_730x272.png)

> 注：computeFocusedWindowLocked：从窗口堆栈从上到下搜索，如果它的宿主App是当前Focused的App，那么它就是Focused窗口。Focused App在窗口切换时已经确定

* **WMS. finishUpdateFocusedWindowAfterAssignLayersLocked**  
  ![](https://box.kancloud.cn/4a1ec3d3ead930dd95394ebf98e9c886_775x141.png)

* **InputMonitor.setInputFocusLw**  
  ![](https://box.kancloud.cn/971646614d40433bf7fa32e95b632c45_681x355.png)

> 注：这里传进来的参数updateInputWindows的值等于false，不会马上调用updateInputWindowsLw来更新窗口，但是接下来新的窗口请求WMS进行layout时，就会调用updateInputWindowsLw来更新窗口

* **InputMonitor. updateInputWindowsLw**
 
  ![](https://box.kancloud.cn/1dfb1e5b6e767b285f8beff853abc625_700x547.png)

> 注： universeBackground：类型为TYPE\_UNIVERSE\_BACKGROUND的窗口  
> TYPE\_UNIVERSE\_BACKGROUND的窗口：Behind the universe of the real windows, in multiuser systems shows on all users’windows

* **InputManagerService.setInputWindows**  
  ![](https://box.kancloud.cn/9726c3e6618505b1c18b83ec3ceb20d7_642x166.png)

* **nativeSetInputWindows**  
  ![](https://box.kancloud.cn/3720113136e034fc41b6cb77aca29b3c_583x103.png)

* **NativeInputManager.setInputWindows**  
  ![](https://box.kancloud.cn/18196f14ddcf6dcabe7dc59c4daaa7c2_724x352.png)

* **InputDispatcher.setInputWindows**  
  ![](https://box.kancloud.cn/16b34ffb4ce46f2ebd1cd76b099d040a_785x480.png)

**注册Client端InputChannel**  
![](https://box.kancloud.cn/30983683431e69dd793148bb464f2e0b_631x413.png)

* \*\*new WindowInputEventReceiver \*\*  
  ![](https://box.kancloud.cn/1c5df2506d93fb233c61fd34ff02e579_680x209.png)

* \*\*new InputEventReceiver \*\*  
  ![](https://box.kancloud.cn/3b8dc3f8a04d3726e0acd7a96ef25294_600x243.png)

* **nativeInit**  
  ![](https://box.kancloud.cn/14d740ea5553e3fec753a9368a6b8545_798x257.png)

* **NativeInputEventReceiver.initialize**  
  ![](https://box.kancloud.cn/043b241cfffd7e7a07ce02dca750e2dc_711x81.png)

> 注：IO事件发生时， NativeInputEventReceriver::handleEvent将被调用

#### **输入事件的分发过程**

**输入事件处理框架**  
![](https://box.kancloud.cn/6320968a871a231e54a2a9664dd39fa5_516x252.png)

**InputReader获得输入事件**

* **InputReader获得输入事件--EventHub.getEvents**  
  ![](https://box.kancloud.cn/2a322d4c8d61c5b198df62cf3ea48945_731x615.png)

* \*\*InputReader获得输入事件 – InputReader.loopOnce \*\*  
  ![](https://box.kancloud.cn/c195f5c187f1d43f2c99f850b5ec0d12_797x561.png)

* \*\*InputReader获得输入事件 – InputReader.processEventsLocked \*\*  
  ![](https://box.kancloud.cn/9d8903fa554a0fec91d13bf84054f744_721x244.png)

* **InputReader获得输入事件 – InputReader.processEventsForDeviceLocked**  
  ![](https://box.kancloud.cn/b81291c54f609df62fbede4689ad9e39_651x278.png)

* **InputReader获得输入事件—InputDevice.process**  
  ![](https://box.kancloud.cn/e95f6a269f890a18b302327348aa790c_725x238.png)

* **InputReader获得输入事件—KeyboardInputMapper.process**  
  ![](https://box.kancloud.cn/50e8e925a99e1a1dc1c4a6d26eaa112f_852x358.png)

* **InputReader获得输入事件—KeyboardInputMapper.processKey**  
  ![](https://box.kancloud.cn/5f9b4bb4546a7b1887b8ac50f52e4c64_766x323.png)

* **InputReader获得输入事件—InputDispatcher.notifyKey**  
  ![](https://box.kancloud.cn/d988910226a1233737cfd68bb40acfc0_594x608.png)

* **InputReader获得输入事件—InputDispatcher. enqueueInboundKeyLocked**  
  ![](https://box.kancloud.cn/fdeed536c67d0b377e7132ca3cbd9ffe_1375x96.png)![](https://box.kancloud.cn/68dc7bb8f1e3734c237024fd73e97303_623x89.png)

**InputDispatcher分发键盘事件**

* **InputDispatcher分发键盘事件 – InputDispatcher. dispatchOnceInnerLocked**  
  ![](https://box.kancloud.cn/e909047f55aade541344ccbac581245b_795x548.png)

* **InputDispatcher分发键盘事件 – InputDispatcher. dispatchKeyLocked**  
  ![](https://box.kancloud.cn/901a340a49d4993ba5ada26a93e000c4_738x496.png)

> 注：  
> HOME键会被PhoneWindowManager的成员函数interceptKeyBeforeDispatching拦载，切换至Home App，这是通过post一个command到InputDispatcher的Command Queue去执行实现的  
> findFocusedWindowTargetsLocked – 会调用checkInjectionPermission来检查当前处理的键盘事件是否是注入的，如果是的话，再检查注入者是否有权限。注入事件的时候也会做权限检查。  
> 注入输入事件：
>
> 1. InputManagerService.injectInputEvent
> 2. InputDispatcher::injectInputEvent
>  
>    findFocusedWindowTargetsLocked还会检查上次分发给Target Window的输入事件是否已经有5s内处理完成，没有处理完成的话就会产生一个ANR Command，并且post到InputDispatcher的Command Queue去执行，最终的ANR窗口是通过mPolicy来通知AMS弹出的

* **InputDispatcher分发键盘事件 – InputDispatcher.dispatchMotionLocked**  
  ![](https://box.kancloud.cn/b231f9a975d0a151ba9e5b1283919758_741x476.png)

* **InputDispatcher分发键盘事件-- InputDispatcher.dispatchEventLocked**  
  ![](https://box.kancloud.cn/23f279a8b7716e21b7e71c48d762eec0_811x244.png)

* **InputDispatcher分发键盘事件-- InputDispatcher.prepareDispatchCycleLocked**

![](https://box.kancloud.cn/e40f04876544e80e276918303046620d_869x491.png)

> 注：标志位的解释参见InputDispatcher.h

* **InputDispatcher分发键盘事件-- InputDispatcher.startDispatchCycleLocked**  
  ![](https://box.kancloud.cn/3359d23e797cc0a518552aae9fc33019_695x573.png)

* **InputDispatcher分发键盘事件—InputPublisher.publishKeyEvent**  
  ![](https://box.kancloud.cn/4c2d06837a09267288577dd6ecdb8910_510x482.png)

* **InputDispatcher分发键盘事件— InputChannel::sendMessage**  
  ![](https://box.kancloud.cn/d6ae9c6839b53ca36871786db2baeec5_679x141.png)

**App获得键盘事件**

* **App获得键盘事件— NativeInputEventReceiver.handleEvent**
 
  ![](https://box.kancloud.cn/14c056aa1fec13ef3033cd4d2f5c04e5_756x126.png)

> 注： 应用程序可以通过InputEventReceiver.nativeConsumeBatchedInputEvents来批量处理InputDispatcher分发过来的输入事件

* **App获得键盘事件—NativeInputEventReceiver.consumeEvents**
 
  ![](https://box.kancloud.cn/2ffb05751c7a7cb445d226c78a81e2a1_842x611.png)

> 注： InputConsumer.consumer的返回值等于WOULD\_BLOCK时表示当前没有发生输入事件，这时候就会开始批量处理刚才缓存起来的输入事件

* **App获得键盘事件—InputComsumer.consume**
 
  ![](https://box.kancloud.cn/1879169f6c0cfe7c0cb3793b860a4ba6_820x665.png)

> 注：AMOTION\_EVENT\_ACTION\_MOVE和AMOTION\_EVENT\_ACTION\_HOVER\_MOVE类型的事件会被缓存起来批量处理

* **App获得键盘事件—InputChannel.receiveMessage**  
  ![](https://box.kancloud.cn/41f5f1967bb53876b424e0ec4a8f3ef4_560x306.png)

* **App获得键盘事件—InputEventReceiver.dispatchInputEvent**  
  ![](https://box.kancloud.cn/834455503da85d52bfbcb3e9a8f8ac72_582x139.png)

* **App获得键盘事件—WindowInputEventReceiver.onInputEvent**  
  ![](https://box.kancloud.cn/ec941002ec5944ddd0e1b6d638e2a143_681x288.png)

* **App获得键盘事件—ViewRootImpl.enqueueInputEvent**  
  ![](https://box.kancloud.cn/427ceef05f71f75478d719f7985fca42_737x524.png)

**App获得键盘事件—ViewRootImpl. scheduleProcessInputEvents**  
![](https://box.kancloud.cn/f90d5f628a40ce5ead2e9ebc5ba4f48f_694x260.png)

* **App获得键盘事件—ViewRootImpl. doProcessInputEvents**  
  ![](https://box.kancloud.cn/230c756d827dcdc54d7bcb77a90061f5_722x295.png)

* **App获得键盘事件—ViewRootImpl. deliverInputEvent**  
  ![](https://box.kancloud.cn/3610528165f0809bef739f9376d349a3_728x410.png)

* **App获得键盘事件—ViewRootImpl. deliverKeyEvent**  
  ![](https://box.kancloud.cn/96ac4ab061ce149fda7e31cd0699eaf1_887x580.png)

* **App获得键盘事件—InputMethodCallback. finishedEvent**  
  ![](https://box.kancloud.cn/156a25656d545f8bf1be4a7d42930e09_869x376.png)

* **App获得键盘事件—ViewRootImpl.dispatchImeFinishedEvent**  
  ![](https://box.kancloud.cn/eeb9ec829e34e3b5271ece567a83d4ab_685x243.png)

* **App获得键盘事件—ViewRootImpl. handleImeFinishedEvent**  
  ![](https://box.kancloud.cn/4ca6c8f8321f0584ff0498565f4f5389_709x560.png)

* **App获得键盘事件—ViewRootImpl. deliverKeyEventPostIme**  
  ![](https://box.kancloud.cn/82f4ef134c42ed4d223dfb6545424601_681x359.png)

* **App获得键盘事件—DecorView. dispatchKeyEvent**  
  ![](https://box.kancloud.cn/7f892afa215bd71ff4f51c834d00ece8_878x531.png)

> 注： getCallback返回的Callback接口指向当前Activity  
> PhoneWindow.onKeyDown和PhoneWindow.onKeyUp会对MENU和BACK等实体键进行处理,对MENU键的处理对应于openPanel的实现，对BACK键的处理对应于closePanel的实现

* **App获得键盘事件—Activity.dispatchKeyEvent**
 
  ![](https://box.kancloud.cn/c2e7157867021e3b15467e9074a6c649_574x344.png)

> 注：getWindow获得的是一个PhoneWindow

* **App获得键盘事件—PhoneWindow.superDispatchKeyEvent**
 
  ![](https://box.kancloud.cn/265ad56e9955facad6c2899bb15d2333_706x606.png)

> 注： DecorView.dispatchKeyEvent将KeyEvent分发给当前获得焦点的View处理

**在App中，依次获得键盘事件的顺序**

* View\(Pre Input Method\)
* Input Method
* View\(Post Input Method\)
* Activity
* Phone Window\(处理MENU、BACK等按键\)
 
  **HOME按键被PhoneWindowManager拦截，直接切换至Home App**

**TextView、输入法和输入法管理器的关系**  
![](https://box.kancloud.cn/544fa15a79c6a20aa485a73ec5304bb7_637x315.png)  
**输入法通过InputConnection.commitText分发过来的字符被封装成一个类型为FLAG\_DELIVER\_POST\_IME的KeyEvent**  
**在ViewRootImpl中，类型为FLAG\_DELIVER\_POST\_IME的KeyEvent不用经过输入法处理，而直接通过deliverKeyEventPostIme分发给View Hierarchy处理**  
**deliverKeyEventPostIme的处理过程与实体 键经过输入法处理后的过程是一样的**

* View
* Activity
* Phone Window



