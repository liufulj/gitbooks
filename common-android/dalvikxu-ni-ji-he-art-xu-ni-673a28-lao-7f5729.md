### **概述**

Android应用程序是运行在Dalvik虚拟机里面的，并且每一个应用程序对应有一个单独的Dalvik虚拟机实例。Android应用程序中的Dalvik虚拟机实例实际上是从Zygote进程的地址空间拷贝而来的，这样就可以加快Android应用程序的启动速度。Dalvik虚拟机与Java虚拟机共享有差不多的特性，例如，它们都是解释执行，并且支持即时编译（JIT）、垃圾收集（GC）、Java本地方法调用（JNI）和Java远程调试协议（JDWP）等，差别在于两者执行的指令集是不一样的，并且前者的指令集是基本寄存器的，而后者的指令集是基于堆栈的。  
本系列文章主要将Dalvik虚拟机的内存管理、垃圾收集、即时编译、Java本地调用、进程和线程管理等。理解Dalvik虚拟机的上述实现细节，有助于在运行时修改程序的行为，例如，拦截Java函数的调用。

* Dalvik虚拟机概述
* Dalvik虚拟机的启动过程
* Dalvik虚拟机的运行过程
* JNI函数的注册过程
* Dalvik虚拟机进程
* Dalvik虚拟机线程

#### **Dalvik虚拟机概述**

Dalvik虚拟机由Dan Bornstein开发，名字来源于他的祖先曾经居住过的位于冰岛的同名小渔村  
Dalvik虚拟机起源于Apache Harmony项目，后者是由Apache软件基金会主导的，目标是实现一个独立的、兼容JDK 5的虚拟机，并根据Apache License v2发布

**Dalvik虚拟机与Java虚拟机的区别**  
![](https://box.kancloud.cn/f2d177cf0d59d2b1ca430a4577431559_728x223.png)

* 基于堆栈的Java指令\(1个字节\)和基于寄存器的Dalvik指令\(2、4或者6个字节\)各有优劣
* 一般而言，执行同样的功能， Java虚拟机需要更多的指令（主要是load和store指令），而Dalvik虚拟机需要更多的指令空间
* 需要更多指令意味着要多占用CPU时间，而需要更多指令空间意味着指令缓冲（i-cache）更易失效
* Dalvik虚拟机使用dex（Dalvik Executable）格式的类文件，而Java虚拟机使用class格式的类文件
* 一个dex文件可以包含若干个类，而一个class文件只包括一个类
* 由于一个dex文件可以包含若干个类，因此它可以将各个类中重复的字符串只保存一次，从而节省了空间，适合在内存有限的移动设备使用
* 一般来说，包含有相同类的未压缩dex文件稍小于一个已经压缩的jar文件

**Dex文件的生成**  
![](https://box.kancloud.cn/3fb3949d058324c82ec385245f2fa715_360x216.png)

**Dex文件的优化**

* 将invoke-virtual指令中的method index转换为vtable index – 加快虚函数调用速度
* 将get/put指令中的field index转换为byte offset – 加快实例成员变量访问速度
* 将boolean/byte/char/short变种的get/put指令统一转换为32位的get/put指令 – 减小VM解释器的大小，从而更有效地利用CPU的i-cache
* 将高频调用的函数，例如String.length，转换为inline函数 – 消除函数调用开销
* 移除空函数，例如Object.
  `<`
  `init`
  `>`
  -- 消除空函数调用
* 将可以预先计算的数据进行预处理，例如预先生成VM根据class name查询class的hash table – 节省Dex文件加载时间以及内存占用空间
* 将invoke-virtual指令中的method index转换为vtable index

```
invoke
-
virtual 
{
v1
,
 v2
}
,
 method@BBBB
→
invoke
-
virtual
-
quick 
{
v1
,
v2
}
,
vtable #
0
xhh

```

* 将get/put指令中的field index转换为byte offset

```
iget
-
object v0
,
 v2
,
 field@BBBB
→
iget
-
object
-
quick v0
,
v2
,
[
obj
+
0x100
]
```

备注：  
[http://www.haogongju.net/art/1171347](http://www.haogongju.net/art/1171347)  
[http://mylifewithandroid.blogspot.com/2009/05/about-quick-method-invocation.html](http://mylifewithandroid.blogspot.com/2009/05/about-quick-method-invocation.html)  
[http://source.android.com/devices/tech/dalvik/dex-format.html](http://source.android.com/devices/tech/dalvik/dex-format.html)

**Dex文件的优化时机**

* VM在运行时即时优化，例如使用DexClassLoader动态加载dex文件时。这时候需要指定一个当前进程有写权限的用来保存odex的目录。
* APP安装时由具有root权限的installd优化。这时候优化产生的odex文件保存在特权目录/data/dalvik-cache中。
* 编译时优化。这时候编译出来的jar/apk里面的classes.dex被提取并且优化为classes.odex保存在原jar/apk所在目录，打包在system image中。

**内存管理**

* Java Object Heap
 
  大小受限，16M/24M/32M/48M/…
* Bitmap Memory\(External Memroy\)：
 
  大小计入Java Object Heap
* Native Heap
 
  大小不受限

**Java Object Heap**

* 用来分配Java对象。Dalvik虚拟机在启动的时候，可以通过-Xms和-Xmx选项来指定Java Object Heap的最小值和最大值。
* Java Object Heap的最小和最大默认值为2M和16M。但是厂商会根据手机的配置情况进行调整，例如，G1、Droid、Nexus One和Xoom的Java Object Heap的最大值分别为16M、24M、32M 和48M。
* 通过ActivityManager.getMemoryClass可以获得Dalvik虚拟机的Java Object Heap的最大值。

**Bitmap Memory**

* 用来处理图像。在HoneyComb之前，Bitmap Memory是在Native Heap中分配的，但是这部分内存同样计入Java Object Heap中。这就是为什么我们在调用BitmapFactory相关的接口来处理大图像时，会抛出一个OutOfMemoryError异常的原因：
  **java.lang.OutOfMemoryError: bitmap size exceeds VM budget**
* 在HoneyComb以及更高的版本中，Bitmap Memory就直接是在Java Object Heap中分配了，这样就可以直接接受GC的管理。

**Native Heap**

* 在Native Code中使用malloc等分配出来的内存，这部分内存不受Java Object Heap的大小限制。
* 注意，不要因为Native Heap可以自由使用就滥用，因为滥用Native Heap会导致系统可用内存急剧减少，从而引发系统采取激进的措施来Kill掉某些进程，用来补充可用内存，这样会影响系统体验。

**垃圾收集\(GC\)**

* Step 1: Mark，使用RootSet标记对象引用

* Step 2: Sweep，回收没有被引用的对象  
  **GingerBread之前**

* Stop-the-word，也就是垃圾收集线程在执行的时候，其它的线程都停止

* Full heap collection，也就是一次收集完全部的垃圾

* 一次垃圾收集造成的程序中止时间通常都大于100ms  
  **GingerBread之后**

* Cocurrent，也就是大多数情况下，垃圾收集线程与其它线程是并发执行的

* Partial collection，也就是一次可能只收集一部分垃圾

* 一次垃圾收集造成的程序中止时间通常都小于5ms

* Dalvik虚拟机执行完成一次垃圾收集之后，我们通常可以看到类似以下的日志输出：

```
D
/
dalvikvm
(
9050
)
:
 GC_CONCURRENT freed 
2049
K
,
65
%
 free 
3571
K
/
9991
K
,
 external 
4703
K
/
5261
K
,
 paused 
2
ms
+
2
ms  

```

* GC\_CONCURRENT表示并行GC，2049K表示总共回收的内存，3571K/9991K表示Java Object Heap统计，即在9991K的Java Object Heap中，有3571K是正在使用的，4703K/5261K表示External Memory统计，即在5261K的External Memory中，有4703K是正在使用的，2ms+2ms表示垃圾收集造成的程序中止时间

**即时编译\(JIT\)**

* 从2.2开始支持JIT，并且是可选的，编译时通过WITH\_JIT宏进行控制

* 基于执行路径\(Executing Path\)对热门的代码片断进行优化\(Trace JIT\)，传统的Java虚拟机以Method为单位进行优化\(Method JIT\)

* 可以利用运行时信息进行激进优化，获得比静态编译语言更高的性能，如Lazy Unlocking机制，可以参考《Oracle JRockit: The Definitive Guide》一书

* 实现原理：[http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html](http://blog.reverberate.org/2012/12/hello-jit-world-joy-of-simple-jits.html)

* 支持JDWP（Java Debug Wire Protocol）协议

  * 每一个Dalvik虚拟机进程都都提供有一个端口来供调试器连接
  * DDMS提供有一个转发端口8870，通过它可以同时调试多个Dalvik虚拟机进程
 
    ![](https://box.kancloud.cn/4fd364d735b86c4dd278211657ff7b6a_704x517.png)

#### **Dalvik虚拟机的启动过程**

Dalvik虚拟机由Zygote进程启动，然后再复制到System Server进程和应用程序进程  
![](https://box.kancloud.cn/baf670ce27a801cb8b9417e4ad226777_715x542.png)

* startVM的过程中会创建一个JavaVMExt，并且该JavaVMExt关联有一个JNIInvokeInterface，Native Code通过它来访问Dalvik虚拟机  
  ![](https://box.kancloud.cn/531c80036cbe4e61f54d794ac0cd30d8_453x234.png)

* startVM的过程中还会为当前线程关联有一个JNIEnvExt，并且该JNIEnvExt 关联有一个JNINativeInterface，Native Code通过它来调用Java函数或者访问Java对象  
  ![](https://box.kancloud.cn/ff10442e32c364f9ef78d8e11c9686c3_536x573.png)

**Dalvik虚拟机在Zygote进程启动的过程中，还会进一步预加载Java和Android核心类库以及系统资源**  
![](https://box.kancloud.cn/da0751e0d400de8584e9528513560e71_591x446.png)

**Dalvik虚拟机从Zygote进程复制到System Server进程之后，它们就通过COW\(Copy On Write\)机制共享同一个Dalvik虚拟机实例以及预加载类库和资源**  
![](https://box.kancloud.cn/49425482eb8c883d1aba8f3c07937b22_674x467.png)

**Dalvik虚拟机从Zygote进程复制到应用程序进程之后，它们同样会通过COW\(Copy On Write\)机制共享同一个Dalvik虚拟机实例以及预加载类库和资源**  
![](https://box.kancloud.cn/e5719c2c3313cbc0b2db6d0fb2cce8ca_678x463.png)

#### **Dalvik虚拟机的运行过程**

* Dalvik虚拟机在Zygote进程中启动之后，就会以ZygoteInit.main为入口点开始运行
* Dalvik虚拟机从Zygote进程复制到System Server进程之后，就会以SystemServer.main为入口点开始运行
* Dalvik虚拟机Zygote进程复制到应用程序进程之后，就会以ActivityThread.main为入口点开始运行
* 上述入口点都是通过调用JNINativeInterface接口的成员函数CallStaticVoidMethod来进入的
 
  J
 
  **JNINativeInterface-**
  **&gt;**
  **CallStaticVoidMethod对应的实现为CallStaticVoidMethodV**
 
  ![](https://box.kancloud.cn/d851ea3e8f137fe28184f4d3f08799ee_561x287.png)

**CallStaticVoidMethodV调用dvmCallMethodV**  
![](https://box.kancloud.cn/724b7293849b514df56f99980542bd3d_536x339.png)

**在Dalvik虚拟机中，无论是Java函数，还是Native函数，都是通过Method结构体来描述的**  
![](https://box.kancloud.cn/52380063cc25e7063abb85a917feca2d_702x392.png)

> **备注**：  
> struct Method定义在文件dalvik/vm/oo/Object.h中

**在Dalivk虚拟机中，通过dvmIsNativeMethod判断一个函数是Java函数还是Native函数**  
![](https://box.kancloud.cn/476c92786b73391740068e107e55712e_506x55.png)

> **备注**：  
> dvmIsNativeMethod定义在文件dalvik/vm/oo/Object.h中

**Native函数直接由CPU执行，Java函数由Dalvik虚拟机解释执行，即通过dvmInterpret函数执行**  
![](https://box.kancloud.cn/f6352c5dd6fba37975588cddfc5a8a12_638x546.png)

**Dalvik虚拟机标准解释器：dvmInterpretStd**  
![](https://box.kancloud.cn/90bdda01aaba9f35ca6574e0c81f17ce_816x555.png)

**Invoke-direct指令由函数invokeDirect执行**  
![](https://box.kancloud.cn/438ce91f8254a796fcd8b58e6f320266_577x306.png)

**函数invokeDirect调用 invokeMethod执行**  
![](https://box.kancloud.cn/c216ef647340cd81277574243074f73a_703x496.png)

#### **JNI函数的注册过程**

**JNI函数注册示例 -- ClassWithJni**  
![](https://box.kancloud.cn/e8fb506deddde63b6abfd75ccf04a17f_527x274.png)

**JNI函数注册示例 -- shy\_luo\_jni\_ClassWithJni\_nanosleep**  
![](https://box.kancloud.cn/9216f0a09f730db97a201688550031dd_636x470.png)

**System.loadLibrary**  
![](https://box.kancloud.cn/ac707743e8a629c43ef330d9efb87331_596x235.png)

**Runtime.loadLibrary**  
![](https://box.kancloud.cn/a80eacf18648f5bd3889829c491fb9de_916x404.png)

**Runtime.nativeLoad**  
![](https://box.kancloud.cn/2628da643058b46a8d28107ea27cefcc_590x342.png)

**dvmLoadNativeCode**  
![](https://box.kancloud.cn/2415f740ea4592ec72f31985f214397a_562x606.png)

**JNI\_OnLoad**  
![](https://box.kancloud.cn/9216f0a09f730db97a201688550031dd_636x470.png)

**jniRegisterNativeMethods**  
![](https://box.kancloud.cn/f53bf93adb88378c1132b0320054021c_558x377.png)

**RegisterNatives**  
![](https://box.kancloud.cn/c179b232c6513134a20fa3fceeaf9f2e_524x395.png)

**dvmRegisterJNIMethod**  
![](https://box.kancloud.cn/4fdac5ec86602308b2487a151dc2a1ad_649x335.png)

**dvmUseJNIBridge**  
![](https://box.kancloud.cn/582810a7f0d7432a096d54bd8e72941c_549x200.png)

**dvmSetNativeFunc**  
![](https://box.kancloud.cn/644e6cfd5427b4b5e062edfa3c72757e_502x306.png)

#### **Dalvik虚拟机进程**

* Dalvik虚拟机进程与下层的Linux进程是一一对应的
* 当ActivityManagerService启动一个组件的时候，发现用来运行该组件的应用程序进程不存在，就会请求Zygote进程创建
* Zygote进程通过调用Zygote类的成员函数forkAndSpecialize来创建

**Zygote.forkAndSpecialize**  
![](https://box.kancloud.cn/7065c5a5ef48996876031a108d019f6b_550x145.png)  
![](https://box.kancloud.cn/5183998a4407e76eddfaa23e0427d168_532x215.png)

**forkAndSpecializeCommon**  
![](https://box.kancloud.cn/1cec59f4f3ac05686186ad9b3483c168_611x592.png)

* Dalvik虚拟机线程与下层的Linux线程是一一对应的
* 在Java层中，可以创建一个Thread对象，并且调用该Thread对象的成员函数start来启动一个Dalvik虚拟机线程
* 在Native层中，也可以通过创建一个Thread对象，并且调用该Thread对象的成员函数run来启动一个Dalvik虚拟机线程

**在Java层创建Dalvik虚拟机线程--Thread.start**  
![](https://box.kancloud.cn/59808cbaa43a84f03fcc8f0a5d9c8457_638x270.png)

**VMThread.create**  
![](https://box.kancloud.cn/99bcc1bf044de8fea806b9567a88d58e_582x450.png)

**dvmCreateInterpThread**  
![](https://box.kancloud.cn/ce7ead3b72882ddb9b1954c2b592f2d2_603x433.png)

**线程启动函数：interpThreadStart**  
![](https://box.kancloud.cn/d150406c1fc754cb6aed57aedf43b802_448x225.png)

**dvmCreateJNIEnv**  
![](https://box.kancloud.cn/db77d4a3b68dc83fc13c64f4b438307d_1024x414.png)

**在Native层创建Dalvik虚拟机线程--Thread::run**  
![](https://box.kancloud.cn/d8fd4591e2b9172c7de4cd40b7b0a542_568x322.png)

**createThreadEtc**  
![](https://box.kancloud.cn/7a8cfed02e08746c44a0253658cf9c7f_561x181.png)  
**androidCreateThreadEtc**  
![](https://box.kancloud.cn/7c2ebecc1244f5c7b29f5b6a44fbe40c_541x216.png)

> **注意**，函数指针gCreateThreadFn所指向的函数在Dalvik虚拟机启动时已经被修改为javaCreateThreadEtc

**javaCreateThreadEtc**  
![](https://box.kancloud.cn/2a47b5b212854dbdb3483077d2c1dc9d_602x377.png)

**androidCreateRawThreadEtc**  
![](https://box.kancloud.cn/6d8ac5e9b27007b29810cb5e297a38cb_558x318.png)

**AndroidRuntime::javaThreadShell**  
![](https://box.kancloud.cn/84dc7372916e7590db4f7f3a9337b4be_527x377.png)

**javaAttachThread**  
![](https://box.kancloud.cn/da97b2d4f64c53a8b151fdf8c5375f4b_490x344.png)  
**AttachCurrentThread**  
![](https://box.kancloud.cn/3f623c5e3f980d04ea1bde2d9e27e1cd_536x143.png)  
**attachThread**  
![](https://box.kancloud.cn/839acd089295b8089663275dca612310_567x344.png)  
**dvmAttachCurrentThread**  
![](https://box.kancloud.cn/232421b654ffa20d868da135ee68e218_588x340.png)

