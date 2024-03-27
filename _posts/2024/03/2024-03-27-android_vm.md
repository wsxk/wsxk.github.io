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

为了更加清晰的了解生成的`apk文件里有什么`，我安装了`android stuidio`，小玩一下（<br>