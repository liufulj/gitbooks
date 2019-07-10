# Android源代码开发和调试环境搭建

#### **概述**

Android源代码开发环境与SDK开发环境相比，优势是可以查看和调试系统源代码，包括Java代码和C/C++代码。这对应用开发也是非常有用的，因为在开发中碰到疑难杂症时可以跟踪到系统内部去定位问题。对于涉及到C/C++代码的开发，例如JNI开发和安全相关开发，更加建议在Android源代码开发环境进行，这样就可以利用gdb以及gdbclient工具进行调试。  
主要讲Android源代码下载、编译和运行，以及C/C++、Java代码的调试。

* 开发机器配置
* Android源码下载、编译和运行
* Linux内核源码下载、编译和运行
* Java代码开发和调试
* C/C++代码开发和调试

#### **开发机器配置**

**硬件**

* CPU I5及以上
* 内存16GB及以上
* 硬盘40GB及以上

**软件**

* Ubuntu 10.04 64bit
* Java SDK 1.6
* 其它Package
* [http://source.android.com/source/initializing.html](http://source.android.com/source/initializing.html)

#### **Android源码下载**

```
mkdir ~/bin

PATH=~/bin:$PATH

curl https://dl-ssl.google.com/dl/googlesource/git-repo/repo>~/bin/repo

chmod a+x ~/bin/repo

mkdir WORKING_DIRECTORY

cd WORKING_DIRECTORY

repo init -u https://android.googlesource.com/platform/manifest-b android-4.2_r1

repo sync
```

* [http://source.android.com/source/downloading.html](http://source.android.com/source/downloading.html)

#### **Android源码编译和运行**

```
 . build/envsetup.sh
 
 lunch full-eng
 
 make –j8
 
 emulator 
```

* [http://source.android.com/source/building-running.html](http://source.android.com/source/building-running.html)

#### **Linux内核源码下载、编译和运行**

```
git clone https://android.googlesource.com/kernel/goldfish.git

cd goldfish

git branch –a remotes/origin/android-goldfish-3.4

git checkout remotes/origin/android-goldfish-3.4

export PATH=$(AOSP)/prebuilts/gcc/linux-x86/arm/arm-eabi-4.6/bin:$PATH

export ARCH=arm

export SUBARCH=arm

export CROSS_COMPILE=arm-eabi-

make goldfish_armv7_defconfig

make

emulator –kernel kernel/goldfish/arch/arm/boot/zImage &
```

* [http://source.android.com/source/building-kernels.html](http://source.android.com/source/building-kernels.html)

#### **Java代码开发和调试**

* 安装Eclipse
  `sudo apt-get install eclipse`

* 拷贝.classpath文件
  * `cd /path/to/android/root`
  * `cp development/ide/eclipse/.classpath .`
  * `chmod u+w .classpath`
* 修改Eclipse内存配置
  * 打开/usr/lib/eclipse/eclipse.ini
  * 将-Xms和-Xmx均设置为2048m
* 导入Android源码到Eclipse
  * File &gt; New &gt;Java Project
  * Pick a project name, "android" or anything you like.
  * Select "Create project from existing source", enter the path to your Android root directory, and click Finish.
* 新增App工程到Android源码工程
  * Project &gt;Properties
  * Select "Java Build Path" from the left-hand menu.
  * Choose the "Source" tab.
  * Click "Add Folder..."
  * Add your app's src directory.
  * Click OK.

**使用Eclipse调试Java源码**  
**启动emulator**

* `cd /path/to/android/root`
* `. build/envsetup.sh`
* `lunch 1`
* `emulator `
  &
 
  **打开ddms**
* ddms
* Select a process
 
  **打开eclipse**
* Run &gt; Debug Configuration...
* Right-click "Remote Java Application", select "New".
* Pick a name, i.e. "android-debug" or anything you like.
* Set the "Project" to your project name.
* Keep the Host set to "localhost", but change Port to 8700.
* Click the "Debug" button and you should be all set.

[http://source.android.com/source/using-eclipse.html](http://source.android.com/source/using-eclipse.html)

#### **C/C++代码开发和调试**

**安装Vim**  
sudo apt-get install vim  
**启动Emulato**r

* $ cd /path/to/android/root
* $ . build/envsetup.sh
* $ lunch 1
* $ emulator 
  &
* $ adb shell

**在adb shell中启动gdbserver**  
`root@android:/ gdbserver :5039 --attach <pid>`

**在另外一个终端启动gdbclient**

* $ cd /path/to/android/root

* $ . build/envsetup.sh

* $ lunch 1

* $ adb forward tcp:5039 tcp:5039

* $gdbclient

* [https://github.com/keesj/gomo/wiki/AndroidGdbDebugging](https://github.com/keesj/gomo/wiki/AndroidGdbDebugging)



