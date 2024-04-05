---
layout: post
tags: [Android]
title: "android 加载流程(打包&启动)"
date: 2024-3-27
author: wsxk
comments: true
---

- [前言](#前言)
- [2. Android虚拟机](#2-android虚拟机)
  - [2.1 DVM虚拟机](#21-dvm虚拟机)
    - [2.1.1 DVM与JVM的差异](#211-dvm与jvm的差异)
  - [2.2 ART虚拟机](#22-art虚拟机)
    - [2.2.1 ART和DVM的区别](#221-art和dvm的区别)
- [3. APK打包流程](#3-apk打包流程)
  - [3.1 android studio生成apk](#31-android-studio生成apk)
- [4. Android系统加载流程](#4-android系统加载流程)
  - [4.1 Bootloader](#41-bootloader)
  - [4.2 linux系统启动](#42-linux系统启动)
  - [4.3 init进程](#43-init进程)
  - [4.4 Zygote进程](#44-zygote进程)
  - [4.5 SystemServer进程](#45-systemserver进程)


## 前言<br>
该文章为[https://wsxk.github.io/android/](https://wsxk.github.io/android/)<br>
的后续<br>
虽然不读前面的文章也无所谓(😄)<br>

## 2. Android虚拟机<br>
`Android虚拟机指的是DVM虚拟机和ART虚拟机，都是用来在Android平台上运行java程序的虚拟机`<br>
### 2.1 DVM虚拟机<br>
`Dalvik虚拟机简称Dalvik VM或者DVM`，是Google专门为Android平台开发的虚拟机，它运行在Android运行库中，需要注意的是DVM并不是一个Jvm虚拟机<br>
#### 2.1.1 DVM与JVM的差异<br>
最显著的差异在于**JVM是基于栈的虚拟机，DVM是基于寄存器的虚拟机**<br>
因为这个差异导致如下结果:<br>

> 1. JVM基于栈的读写数据耗费指令更多，更费时；
> 2. DVM基于寄存器的读写数据耗费指令少，速度快，但是因为需要显式指定操作数，单个指令会比jvm的指令要长
> 3. 虽然DVM指令比jvm长，然而jvm需要的指令数目多，因此实际上占据的空间可能差不多

另外这两个虚拟机执行的字节码也有差异<br>
下图是`JVM虚拟机中Java代码从编写到执行的过程`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240327214938.png)
下图是`DVM虚拟机中java代码从编写到执行的过程`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240327215105.png)
**可以看到，其实DVM字节码是从java字节码通过dx/d8工具转换而来**<br>

### 2.2 ART虚拟机<br>
`ART虚拟机`是`Android4.4`发布的，用来替换`Dalvik虚拟机`，Android 4.4默认采用的还是`DVM`，不过系统会提供一个选项来开启ART。在`Android 5.0时，默认采用ART。`<br>
#### 2.2.1 ART和DVM的区别<br>
DVM虚拟机中应用每次运行时，字节码都需要通过即时编译器转换成机器码，这会使应用的运行效率降低<br>
ART与DVM最大的区别是，ART虚拟机在安装apk程序时，对.dex文件进行了一次预编译，并将编译结果保存起来，**这样不用每次运行程序都重新对.dex文件编译成机器码，提高程序的运行速率**<br>

## 3. APK打包流程<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240327221854.png)
简单来说了，一个`android`项目经过**编译和打包**这个过程后，会形成`apk文件`，经过签名后，就可以通过`adb`安装到我们的手机上<br>
下面这种图列出了**从Android项目到apk的全过程**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240327232556.png)


| 名称 | 功能介绍 |  操作系统中的路径 |
|:------:|:------:|:------:|
|   aapt   |   Android资源打包工具，并生成R.java和resources.arsc文件   |   ${ANDROID_SDK_HOME}/build-tools/appt   |
|   aidl   |   Android接口描述语言.aidl文件转化为.java文件的工具   |    ${ANDROID_SDK_HOME}/build-tools/aidl   |
|   java compiler(javac)   |   把java文件转换为class文件   |   java安装路径   |
|   dex(d8)  |   将项目生成的class文件与第三方库的class文件结合起来并生成dex文件   |   ${ANDROID_SDK_HOME}/build-tools/d8.bat  |
|   apkbuilder(gradle)   |   将资源文件和.dex文件生成未签名的.apk文件   |   zip命令  |
|   jarsigner   |   对未签名的apk进行签名   |   ${ANDROID_SDK_HOME}/build-tools/apksigner.bat  |
|   zipalign(release mode)   |   字节码对齐工具   |   ${ANDROID_SDK_HOME}/build-tools/zipalign.exe |

整个apk打包流程为：<br>
> 1. **使用aapt来打包res资源文件，生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）**
> > * res目录，有9种子目录
> > * R.java文件。里面拥有很多个静态内部类，比如layout，string等。每当有这种资源添加时，就在R.java文件中添加一条静态内部类里的静态常量类成员，且所有成员都是int类型。
> > * resources.arsc文件。这个文件记录了所有的应用程序资源目录的信息，包括每一个资源名称、类型、值、ID以及所配置的维度信息。我们可以将这个文件想象成是一个资源索引表，这个资源索引表在给定资源ID和设备配置信息的情况下，能够在应用程序的资源目录中快速地找到最匹配的资源。
> 2. **AIDL，Android接口定义语言，Android提供的IPC的一种独特实现。这个阶段处理.aidl文件，生成对应的Java接口文件。**
> 3. **通过Java Compiler编译R.java、Java接口文件、Java源文件，生成.class文件。**
> 4. **通过d8.bat命令，将.class文件和第三方库中的.class文件处理生成classes.dex。**
> 5. **将 classes.dex，resources.arsc，res文件夹(res/raw资源被原装不动地打包进APK之外，其它的资源都会被编译或者处理)、Other Resources(assets文件夹)，AndroidManifest.xml打包成apk文件。**
> 6. **对apk进行签名，可以进行Debug和Release 签名。**
> 7. **release mode 下使用 aipalign 进行align，即对签名后的apk进行对齐处理。**
> > Zipalign是一个android平台上整理APK文件的工具，它对apk中未压缩的数据进行4字节对齐，对齐后就可以使用mmap函数读取文件，可以像读取内存一样对普通文件进行操作。如果没有4字节对齐，就必须显式的读取，这样比较缓慢并且会耗费额外的内存。
> > 在 Android SDK 中包含一个名为 zipalign 的工具，它能够对打包后的 app 进行优化。 其位于 SDK 的 \build-tools\34.0.0\zipalign.exe 目录下

### 3.1 android studio生成apk<br>
为了更加清晰的了解生成的`apk文件里有什么`，这里列出一些常见的目录和文件<br>

| 文件/目录 | 功能介绍 |
|:------:|:------:|
|assets| 目录中含有的是apk运行需要的静态文件 |
|lib| 目录中存放android程序会调用的so|
|META-INF|目录中存放的是签名信息（如果有）以及各个组件的版本号|
|res| 目录中存放了资源文件（图片，文本，xml布局，etc）|
|AndroidManifest.xml|清单，描述了apk的程序入口，版本，等等信息|
|classes.dex| 先前提到的smali构成的文件|
|resources.arsc| 资源名称和对应的id关系，R.java中会保存所有资源的id，apk程序如果使用到了某个资源，会使用R.java中的资源id，在resources.arsc中找到对应的资源名称，并索引该资源|


## 4. Android系统加载流程<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240328230114.png)
该图引用自[https://www.jianshu.com/p/9f978d57c683](https://www.jianshu.com/p/9f978d57c683)<br>
我认为能比较好的描述`Android系统的加载流程`
### 4.1 Bootloader<br>
一般开机，即手机开始供电，运行的程序就是`bootloader，其起始地址是固定的，取决于各个厂商`<br>
**bootloader的作用就是将 系统的软硬件环境带到一个合适的状态，为运行操作系统做好准备**<br>
`bootloader`第一个装载的是`fastboot`<br>
> 1. 当fastboot被装载后便开始运行，它一般会先检测用户是否按下某些特别按键，这些特别按键是fastboot在编译时预先被约定好的，用于进入调试模式。如果用户没有按这些特别的按键，则fastboot会从NAND Flash中装载Linux内核，装载的地址是在编译fastboot时预先约定好的。
> 2. 如果你试过root android手机(如： pixel 2)，可以知道刷机时需要在fastboot的时候通过电脑把android镜像刷到手机中
> 3. fastboot的作用是初始化硬件环境，比如网口、SDRAM等等

### 4.2 linux系统启动<br>
`fastboot会装载linux内核`<br>
插播一个题外话，如果你装过linux系统，你会知道，实际上要安装是一个名为`zImage`的文件<br>

```
zImage是一个压缩后的linux内核，包括以下几个部分：
1. head.o：是内核的头部文件，负责初始设置
2. misc.0：含有负责解压内核的代码
3. piggy.gzip.o： 使用gzip算法压缩的内核镜像代码

zImage是由vmlinux压缩得到的：
vmlinux 是 Linux 内核的一个重要文件，它是内核的未压缩可执行版本。
vmlinux 是在内核编译过程中生成的，包含了内核的全部代码和数据。
这个文件是 ELF（Executable and Linkable Format）格式的，这是一种常用的可执行文件和共享库的格式。
vmlinux对于调试来说是 非常非常非常 重要的！！！😀
```

回到正题，**1. 装载linux内核的第一件事就是对内核镜像进行解压缩，并放到内存中**<br>
**2. 第二阶段是进入内核的入口函数:start_kernel(),它主要完成剩余与硬件平台的相关初始化工作（包括设置体系结构相关的环境，初始化内存结构，建立MMU，页表 等等），在进行一些系列的与内核相关的初始化步骤后，调用第一个用户进程——init进程并等待用户进程的执行**<br>
`linux在启动init进程前的最后一个步骤是挂载根文件系统（只读，此时linux还在启动阶段，并不稳定，设置只读是为了保证即使linux宕机了也不会损坏根牡蛎），启动init必须要有根文件系统`，其至少要有如下几个目录：<br>

> * /etc/：存储重要的配置文件
> * /bin/：存储常用且开机时必须用到的执行文件。
> * /sbin/：存储着开机过程中所需的系统执行文件。
> * /lib/：存储/bin/及/sbin/的执行文件所需要的链接库，以及Linux的内核模块
> * /dev/：存储设备文件

之所以要挂在根文件系统，是因为**需要安装适当的内核模块，以便驱动硬件设备,而init程序和内核模块(.ko)都存放在根目录中，没想到吧（**<br>
第三个阶段就是**3. init程序启动 init服务负责后续初始化系统使用环境的工作,且之后如果有用户程序需要启动，都由init派生**<br>

### 4.3 init进程<br>
inti进程作为第一个被内核启动的用户空间进程，主要做如下几个动作：<br>

> 1. 初始化系统环境：init 进程负责设置系统环境，包括挂载文件系统、设置网络配置、初始化设备节点等。
> 2. 启动守护进程：init 进程会根据配置文件（如 /init.rc 和其他 .rc 文件）启动一系列的守护进程（daemons），这些进程负责系统的各种服务和功能，例如 logd（日志服务）、servicemanager（服务管理器）、surfaceflinger（显示系统服务）等。
> 3. 启动 Zygote 进程：init 进程会启动 Zygote 进程，它是 Android 应用程序和系统服务的基础。Zygote 进程加载 Java 虚拟机（ART 或 Dalvik）和预加载的类和资源，以便加快应用程序的启动速度。所有的 Android 应用程序都是通过从 Zygote 进程 fork 出来的，共享相同的虚拟机实例和预加载的资源。

需要详细了解代码的话，有高人相助[https://www.jianshu.com/p/4e5909d24d65](https://www.jianshu.com/p/4e5909d24d65)<br>

### 4.4 Zygote进程<br>
先前说过，所有的Android程序均有Zygote进程生成，但是**Zygote跟传统的linux加载程序方式不同，如下图所示**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240331143245.png)
`传统linux都是通过fork+exec来启动程序， Zygote进程fork出来后，因为要执行的Dalvik的字节码，其他java环境的需要是一致的，所以只需要加载apk中的字节码解释执行即可，不需要通过exec来执行`<br>
**通过这种方式来减少加载java环境的时间**<br>
了解详情[https://www.jianshu.com/p/4e5909d24d65](https://www.jianshu.com/p/4e5909d24d65)<br>

### 4.5 SystemServer进程<br>
systemserver进程由Zygote进程fork而来，其实该进程主要**承接apk应用的所有服务需要，看图就知道了**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240331145259.png)
解释：**1. Android 系统中，Launcher 进程是指运行主屏幕（Home Screen）应用程序的进程,由Zygote进程生成**<br>
**2. Binder是android提供的一种IPC通信机制(作为备选项，android的IPC通信机制以后在有需要时继续研究)**<br>
**3. 总而言之，当我们触摸屏幕，点击某个apk，其实都是在和launcher进程进行交互，当要启动某个apk时，launcher会往system_server发送请求，system_server将请求转发给Zygote进程，Zygote进程fork新进程后，不再参与新进程任何后续应用的使用，相应的使用需求都需要通过System_server来解决**<br>
