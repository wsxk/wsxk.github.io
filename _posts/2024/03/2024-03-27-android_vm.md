---
layout: post
tags: [Android]
title: "android 加载流程"
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