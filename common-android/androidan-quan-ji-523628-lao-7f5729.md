### **概述**

Android应用程序是运行在一个沙箱中。这个沙箱是基于Linux内核提供的用户ID（UID）和用户组ID（GID）来实现的。Android应用程序在安装的过程中，安装服务PackageManagerService会为它们分配一个唯一的UID和GID，以及根据应用程序所申请的权限，赋予其它的GID。有了这些UID和GID之后，应用程序就只能限访问特定的文件，一般就是只能访问自己创建的文件。此外，Android应用程序在调用敏感的API时，系统检查它在安装的时候会没有申请相应的权限。如果没有申请的话，那么访问也会被拒绝。对于有root权限的应用程序，则不受上述沙箱限制。此外，有root权限的应用程序，还可以通过Linux的ptrace注入到其它应用程序进程，以及系统进程，进行各种函数调用拦截。

本系列主要讲代码加壳、注入和拦截技术的，包括：

1. SO注入。也就是从一个进程向另外一个进程注入一个SO文件，通过该注入的SO文件就可以实现函数拦截功能。
2. SO加壳。加壳的目的自然就是加大别人对自己的C/C++代码进行静态逆向难度了，这个技术的关键是要实现一个能纯内存操作的Linker了。也就是说，解密后的SO文件内容是保存在一个内存缓冲区的，然后再针对该内存缓冲区进行解析和链接，最终形成一段可执行的代码。这个过程不会产生任何文件供别人做静态分析。
3. C/C++函数GOT拦截。通过修改SO的GOT项来实现函数拦截。这个技术的特点是简单和稳定，但是不足之处于它是针对函数的调用方进行拦截的，而不是针对函数本身的实现来进行拦截的。这样当我们想对某一个函数进行拦截的时候，就必须要检查进程内所有的模块，然后对调用了目标函数的模块的相关GOT 项进行修改。此外，如果某一个模块是通过动态SO加载技术（dlopen、dlsym）来调用目标函数的话，GOT拦截就失效了，因为动态SO加载技术不会产生GOT项。
4. C/C++函数INLINE拦截。这种方法是直接对目标函数的前面几条指令进行修改，用来实现拦截技术。INLINE拦截没有上述GOT拦截的缺点，但是它的实现会复杂很多。由于绝大部分Android设备都是基于ARM架构，因此这里只讨论ARM架构的C/C++函数INLINE拦截。ARMl架构主要分为ARM和THUMB两种指令集，也就是在Android设备上运行的C/C++函数分为ARM和THUMB两种类型。对于ARM指令集的函数，对它们进行拦截至少需要修改头8个字节；对于THUMB指令集，对它们进行拦截至少需要修改头12个字节。无论ARM指令还是THUMB指令函数，我们要修改的头8个字节或者12个字节都很容易碰到跳转或者PC相对寻址指令，这样就需要对指令进行重定位。这个重定位工作相当于繁重和麻烦，得实现一个ARM和THUMB指令解析库才行。不像X86的函数INLINE拦截，只需要函数的头5个字节即可，而且这5个字节几乎都是堆栈相关的操作，不会涉及到跳转或者PC相对寻址指令。
5. DEX注入。在SO注入的基础上，要对目标进程进行DEX注入是相当简单的，通过DexClassLoader即可实现。
6. DEX加壳。DEX加壳与SO加壳一样，都要求在解密之后，能够进行纯内存操作，中间不要产生任何和DEX或者ODEX文件，否则的话，就会给别提供静态分析的机会，这样就失去了加壳的目的。
7. Java函数拦截。与C/C++函数拦截相对，Java函数拦截要优雅得多，因为所有的Java函数都是通过虚拟机来执行的。Dalvik虚拟机执行的函数分为Java和Native两种，它们都是使用Method结构体来描述。当一个Method结构体描述的是一个Java函数时，它有一个成员变量就指向该Java函数的方法区。而当一个Method结构体描述的是一个Native函数，它有一个成员变量指向该Native函数的地址。因此，主要我们能将一个用来描述Java函数的Method结构体修改为一个指向Native函数的Method结构体，就可以骗过Dalvik虚拟机来执行我们所指定的Native函数，从而实现拦截。

以上7个技术点涵盖了Android安全的攻与防基础。在这些基础上不仅可以保护我们自己的代码，还可以对别人的代码进行攻击。

* Android安全模型
* SO注入技术
* SO加壳技术
* C/C++函数拦截技术
* DEX注入技术
* DEX加壳技术
* Java函数拦截技术

#### **Android安全模型**

![](https://box.kancloud.cn/3f0d2b143607df6f07356a8b039f7263_394x217.png)  
![](https://box.kancloud.cn/78b0f90cd60a9caa3b6dcba231347c41_619x372.png)

**用户**

* 系统中可以存在多个用户，每一个用户都具有一个UID
* 用户按组划分形成用户组，每一个用户组都具有一个GID
* 一个用户可以属于多个用户组

**文件**

* 每一个文件都具有三种权限
 
  Read、Write、Execute
* 文件权限按用户属性分为三组
 
  Owner、Group、Other

![](https://box.kancloud.cn/faced38fa41365815c90cc76f50fc0f1_827x544.png)

**进程**

* UID -- setuid
* GID -- setgid
* Supplementary GIDS – setgroups
* Capabilities -- capset

![](https://box.kancloud.cn/68f5dbba5a1568e18ecd86b3b3ca1f90_795x537.png)

* 系统中的第一个进程Init的UID是root
* 子进程的UID默认与父进程相同，但可以通过setuid进行修改
* 子进程被fork之后exec了一个设置了SUID位的bin文件，那么子进程的UID变为该bin文件的Ower UID
 
  ![](https://box.kancloud.cn/c076c4303308e2baaa9f7f1199dfcb79_727x181.png)

![](https://box.kancloud.cn/3460d11085d6fde58a32b7bd323b88f9_612x275.png)

**每一个APK在安装的时候，PMS都会给它分配一个唯一的UID和GID**

* 如果两个APK具有相同的签名，那么可以通过android:sharedUserId申请分配相同的UID和GID
* 如果一个APK具有平台签名，那么可以通过android:sharedUserId=“android.uid.system”获得System UID

> **备注**  
> 通过Master Key漏洞获得System UID  
> [http://drops.wooyun.org/papers/219](http://drops.wooyun.org/papers/219)  
> [http://safe.baidu.com/2013-10/android-masterkey-9695860.html](http://safe.baidu.com/2013-10/android-masterkey-9695860.html)  
> [http://safe.baidu.com/2013-11/masterkey-9950697.html](http://safe.baidu.com/2013-11/masterkey-9950697.html)  
> 打开文件/data/local.prop，设置以下属性：  
> `ro.kernel.qemu=1`  
> 即可使得adb具有root权限

**每一个APK都可以通过申请若干个Permission**

* 有些Permission需要具有平台签名才可以申请，如INSTALL\_PACKAGES

**每一个Permission都对应于一个Supplementary GID，因此，给APK分配Permission即为APK分配Supplementary GID**  
[http://developer.android.com/reference/android/Manifest.permission.html](http://developer.android.com/reference/android/Manifest.permission.html)

**APK进程是由UID为root的Zygote进程fork出来的，fork之后**：  
![](https://box.kancloud.cn/0023bc63d94db2a1bcc838123965dc26_800x482.png)

[http://man7.org/linux/man-pages/man2/capset.2.html](http://man7.org/linux/man-pages/man2/capset.2.html)

**PMS记录有每一个APK所申请的Permission，当APK调用敏感API时，相应的模块就会通过PMS会验证调用APK是否申请有相应的Permission**  
![](https://box.kancloud.cn/834798c84859e17f646c1c85b2ea4b7d_774x305.png)

**突破沙箱**  
![](https://box.kancloud.cn/5af0468ae7e75ad25a8d716735f0db75_729x458.png)

* **突破沙箱：创建其它APK也能访问的文件**  
  ![](https://box.kancloud.cn/c9ce31d3ded93d312900c02d50b28a3b_1036x165.png)

* **突破沙箱：Binder IPC**  
  ![](https://box.kancloud.cn/2de0c8df1c411f3bafa2f1fdd6b76ad9_781x450.png)

* **突破沙箱：Content Provider**  
  ![](https://box.kancloud.cn/d2ac0f1a190b62a13dd2ec6be4b3fce7_761x337.png)

* **突破沙箱：黑客技术**

  * SO注入

  * C/C++函数拦截
  * DEX注入
  * Java函数拦截
  * ……

#### **SO注入技术**

**ptrace**

```
long 
ptrace
(
enum __ptrace_request request
,
 pid_t pid
,
 void 
*
addr
,
 void 
*
data
)
;
```

[http://man7.org/linux/man-pages/man2/ptrace.2.html](http://man7.org/linux/man-pages/man2/ptrace.2.html)

![](https://box.kancloud.cn/09d5249854ddae50433a3362cc896f9e_457x361.png)

* Step 1: PTRACE\_ATTACH到目标进程，并且让目标进程发生PTRACE\_SYSCALL时停止
* Step 2: PTRACE\_GETREGS保存目标进程的上下文
* Step 3: PTRACE\_SETREGS改写目标进程的PC寄存器，使得它指向函数mmap的地址
* Step 4: PTRACE\_CONT让目标进程恢复执行，这时候将会执行函数mmap
* Step 5: PTRACE\_GETREGS获得目标进程的R0寄存器值，即为函数mmap的返回值，指向在目标进程地址空间分配的一块内存
* Step 6: PTRACE\_POKETEXT往在目标进程分配的地址写入以下一段SHELL CODE
 
  ![](https://box.kancloud.cn/be4badc25d1590f051a2f94d806ef01f_813x444.png)
* Step 7: PTRACE\_SETREGS改写目标进程的PC寄存器，使得它指向上述SHELL CODE的起始地址\_inject\_start\_s
* Step 8: PTRACE\_DETTACH目标进程，目标进程恢复执行后，就会执行注入的
* Step 9: 注入的SHELL CODE在目标进程加载一个SO，并且找到这个SO的指定入口函数，进行调用

#### **SO加壳技术**

**系统中的SO文件由一个叫Linker的加载器负责加载，即调用dlopen函数进行加载**：

```
void 
*
dlopen
(
const char 
*
filename
,
 int flag
)
;
```

如果能将第一个参数改为一个内存地址，就可以实现从内存加载SO的功能，进而可以对该内存进行加密处理

**dlopen**  
![](https://box.kancloud.cn/e1d5d4523c41d7808cedcd50281e4d6d_491x287.png)

**find\_library**  
![](https://box.kancloud.cn/158ec1e42211b5b99910794c52716654_591x350.png)  
**load\_library**  
![](https://box.kancloud.cn/ff95d6318446480d7ce76a0cd5060057_648x507.png)

**Read Elf32\_Ehdr**  
Elf32\_Ehdr header\[1\];  
read\(fd.fd, \(void\*\)header, sizeof\(header\)\)

[http://man7.org/linux/man-pages/man5/elf.5.html](http://man7.org/linux/man-pages/man5/elf.5.html)

**ReadElf32\_Phdr**  
![](https://box.kancloud.cn/06c02819506c1be5cbd1328b90d13c44_804x572.png)

**Reserve Enough Memory**  
![](https://box.kancloud.cn/3599579b0cfc31c2b003940b0afbda76_711x553.png)

**Load Segments**  
![](https://box.kancloud.cn/f0045c85294ad668e391d76572e640cc_716x666.png)

**What data we need?**  
Elf32\_Ehdr  
Elf32\_Phdr  
Segments  
\*\*How to fill above data? \*\*  
![](https://box.kancloud.cn/dee65aff7e714622cfa6ddfa8ecfa17c_549x171.jpg)

**struct elfinfo**  
![](https://box.kancloud.cn/2136f001c8f856f3546c373ee924d3a7_332x177.png)

**Open file and create elfinfo**  
![](https://box.kancloud.cn/cf4dba1ac2298f57323fbde85ef492f3_512x220.png)

**Read Elf32\_Ehdr**  
![](https://box.kancloud.cn/12ec737911227705d55e23f66fdcc08e_609x145.png)

**Read Elf32\_Phdr**  
![](https://box.kancloud.cn/0246198dbe2a1449c4a5fc1e636c4b9e_762x183.png)

**Read segments**  
![](https://box.kancloud.cn/f5664493e5011e07da07970c21e24766_663x344.png)

#### **C/C++函数拦截技术**

* Got Hook
* VTable Hook
* Inline Hook

**Got Hook**  
![](https://box.kancloud.cn/7a29bba9054d7e599dd567bd24741dd3_888x543.png)

* **Step 1: Find the address of eglSwapBuffers**

```
void 
*
 handle 
=
dlopen
(
“
/
system
/
lib
/
libEGL
.
so”
,
 RTLD_NOW
)

void
*
 addr 
=
dlsym
(
handle
,
 “eglSwapBuffers”
)
;
```

* \*\*Step 2: Find the .got section \*\*  
  ![](https://box.kancloud.cn/629714d0b4e95b381ff7713270b0bba5_740x278.png)

* **Step 3: Find the address of eglSwapBuffers in .got section**  
  ![](https://box.kancloud.cn/00d47b31d69f77abc5c6f37024cb6654_648x176.png)

* **Step 4: Replace it**  
  ![](https://box.kancloud.cn/cf46eaf7b5690340277f5ea60b6f217a_797x251.png)

**VTable Hook**  
![](https://box.kancloud.cn/2e28e9cdcfc42ebf07635fecb79b1594_896x562.png)

* \*\*Step 1: Find the address of Surface::unlockAndPost \*\*

```
void 
*
 handle 
=
dlopen
(
“
/
system
/
lib
/
libgui
.
so”
,
 RTLD_NOW
)

void
*
 addr 
=
dlsym
(
handle
,
“_ZN7android7Surface13unlockAndPostEv”
)
;
```

* **Step 2: Find the .data.rel.ro section**  
  ![](https://box.kancloud.cn/8ef459c80d2856362c353693fbbda594_759x265.png)

* \*\*Step 3: Find the address of Surface::unlockAndPost in .data.rel.ro section \*\*  
  ![](https://box.kancloud.cn/5c65328ddf355cabefb5f1f3ed667b55_764x175.png)、

* **Step 4: Replace it**  
  ![](https://box.kancloud.cn/cf46eaf7b5690340277f5ea60b6f217a_797x251.png)

**Inline Hook**  
![](https://box.kancloud.cn/809c5fc44df0208e984e1ea848549a0d_648x534.png)

* **Problem**
 
  ![](https://box.kancloud.cn/8cd16754795b884afcca7451f41f2956_497x338.png)
  移动的指令可能包含：
 
  **普通指令**
* PC相对寻址指令
* 跳转指令

**对于PC相对寻址和跳转指令**：

* 需要进行重定位

**因此，实现Inline Hook要求**：

* 动态的指令解析
* 动态的指令重定位

#### **DEX注入技术**

![](https://box.kancloud.cn/aec648f67b895c0f98d30e85a844b481_448x414.png)  
![](https://box.kancloud.cn/47d375ffab1c3cabcb7e2ca74033e988_1190x712.png)

#### **DEX加壳技术**

* 通过DexClassLoader可以动态地加载DEX文件，但是它在加载DEX文件的过程会生成一个ODEX文件，给别人提供了静态逆向的可能

* 通过分析DexClassLoader的实现可以知道，它是通过DexFile来实现动态加载DEX文件的

* 进一步分析DexFile的实现，发现它提供了两个隐藏接口来实现加载内存DEX文件  
  ![](https://box.kancloud.cn/b92912095ecae222f452bd3f81ce806b_891x269.png)  
  ![](https://box.kancloud.cn/efd4bf1705eb13d63cdccfb5bb26b238_764x706.png)

* 但是，DexFile从Android 4.0开始才支持加载内存DEX文件，如何支持Android 4.0以下的版本呢？

* 通过分析DexFile加载内存DEX文件的实现可以发现，里面用到的关键函数都可以从libdex.a和libdvm.so获得

* 于是，可以模仿Android 4.0，实现DexFile加载内存DEX文件的功能

* 辅助数据结构和函数  
  ![](https://box.kancloud.cn/05c4442c3376dbe3783ca2f73b5a1bf8_737x407.png)

* **Step 1: custome\_load\_class\_from\_memory**  
  ![](https://box.kancloud.cn/41f7c8b0697298433d1a7002d9b0d4f7_1015x656.png)

* **Step 2: open\_dex\_from\_memory**  
  ![](https://box.kancloud.cn/e15585307e794cb70d4a207cc2c76669_757x621.png)

* **Step 3: raw\_dex\_file\_open\_array**  
  ![](https://box.kancloud.cn/9a151f0b49a17c3787de87c74a8c9db7_773x309.png)

* **Step 4: prepare\_dex\_in\_memory**  
  ![](https://box.kancloud.cn/3be6a7c817c52387dd6ec9dbeccc7fed_681x275.png)

* \*\*Step 5: rewrite\_dex \*\*  
  ![](https://box.kancloud.cn/51047ef47a51d0e13403c3e7c234c7fa_751x654.png)

* **Step 6: dex\_file\_open\_partial**  
  ![](https://box.kancloud.cn/3d95f6be35110f409f567f32d92364e8_743x585.png)

* **Step 7: allocate\_aux\_structures**  
  ![](https://box.kancloud.cn/6af58febdd752660c13c9296a02a5dae_672x691.png)

* **Step 8: add\_to\_dex\_file\_table**  
  ![](https://box.kancloud.cn/b3080bd253a04fdbaa9331dec3acb140_711x469.png)

* 上述过程用到的关键函数均可从libdex.a和libdvm.so获得：

  * dexSwapAndVeriry
  * dexCreateClassLookup
  * dexFileParse
  * dvmAllocRegion
  * dvmAllocAtomicCache
  * dvmHashTableLock
  * dvmHashTableLookup
  * dvmHashTableUnlock

#### **Java函数拦截技术**

* 在Dalvik虚拟机中，无论是Java函数，还是Native函数，都是通过Method结构体来描述的  
  ![](https://box.kancloud.cn/52380063cc25e7063abb85a917feca2d_702x392.png)

* Dalvik虚拟机通过dvmIsNativeMethod判断一个函数是Java函数还是Native函数  
  ![](https://box.kancloud.cn/476c92786b73391740068e107e55712e_506x55.png)

* Dalvik虚拟调用一个函数之前，首先判断它是Java函数还是Native函数  
  ![](https://box.kancloud.cn/9b04b1597e470c5276aa6ff94e7a6f94_702x496.png)

* Java函数拦截技术原理分析

  * 对于Java函数，Davik虚拟机使用解释器来执行
  * 对于Native函数， Davik虚拟机找到它的函数指针nativeFunc，进行直接调用
  * 如果我们能把一个Java函数修改为Native函数，并且将nativeFunc指针设置为自定义的函数，那么就可以实现拦截了
  * 拦截完成之后，根据情况决定是否需要调用原来的Java函数，即可完成整个拦截过程

* libdvm导出了两个函数dvmDecodeIndirectRef和dvmSlotToMethod，如果我们知道一个Java函数在它所属的Class里面的位置Slot，那么就可以通过它们获得该Java函数在Dalvik虚拟内部所对应的Method结构体：  
  ![](https://box.kancloud.cn/812850146e131832da2197f1c68525b4_1078x140.png)

* 得到一个Java函数在Dalvik虚拟内部所对应的Method结构体之后，就可以将它设置为Native函数：  
  ![](https://box.kancloud.cn/ed432dc43769c8cf4b57b49e83cfe3a1_531x111.png)

**如何获得一个Java函数所属的Class，以及它在该Class的位置Slot呢？**  
**假设我们知道：**

* Java函数的名称—methodName
* Java函数的原型—prototype
* Java函数的类名称—className
* 用来加载该Java类的ClassLoader--classLoader

**Step 1: 获得Class对象**

```
Class
<
?
>
 clazz 
=
 classLoader
.
loadClass
(
className
)
;
```

**Step 2: 获得Method对象**

```
Method method 
=
 clazz
.
getDeclaredMethod
(
methodName
,
 prototype
)
;
```

**Step 3: 获得clazz的Slot域描述**

```
Field field 
=
 clazz
.
getDeclaredField
(
“Slot”
)
;
```

**Step 4: 获得method的slot**

```
int slot
=
 field
.
getInt
(
method
)
;
```

**Step 5: 将clazz和slot通过JNI传递到C/C++层，调用dvmDecodeIndirectRef和dvmSlotToMethod**

```
ClassObject
*
 declared_classs 
=
(
ClassObject
*
)
dvmDecodeIndirectRef
(
dvmThreadSelf
(
)
,
 clazz
)
;

Method
*
 method 
=
dvmSlotToMethod
(
declared_class
,
 slot
)
;
```



