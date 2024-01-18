---
layout: post
tags: [fuzz]
title: "IOT fuzz 面向漏洞检测的物联网设备辅助异常与崩溃捕获工具的研究"
author: wsxk
date: 2023-2-9
comments: false
---

- [IOT fuzz 现状](#iot-fuzz-现状)
- [解决方案](#解决方案)
  - [1. Static Instrumentation](#1-static-instrumentation)
  - [2. Binary Rewriting](#2-binary-rewriting)
  - [3.Physical Re-Hosting](#3physical-re-hosting)
  - [4.Full Emulation](#4full-emulation)
  - [5.Partial Emulation](#5partial-emulation)
  - [6.Hardware-Supported Instrumentation](#6hardware-supported-instrumentation)
- [题外话](#题外话)


## IOT fuzz 现状<br>
根据先前的 [iot安全背景知识](https://wsxk.github.io/iot%E5%85%A5%E9%97%A8%E4%B8%80/),我们总结过，嵌入式设备大概分为I型，II型，III三型，其中I型和II型比较接近现有的传统`PC计算机`，起码是有分为`内核态和用户态`的，换句话说，`I型和II型嵌入式固件多少是有检测机制的(segment fault ,etc)`，这些设备其实可以用传统的`fuzz`技术来进行测试。<br>
然而，对于`III型嵌入式固件(monolithic firmware)`而言，它是没有分成内核和用户态的，所有的程序都集成到了一个程序当中。这类程序通常没有相应的检测机制（节省空间），安装这种固件的嵌入式设备，通常资源都是有限的，对于fuzz来说，无疑是晴天霹雳。<br>
> 1. 资源有限导致fuzz效率很差
> 2. 没有检测机制导致fuzz无法捕捉到相应的错误
> > 具体而言，fuzz依靠Observing exit status（查看返回的错误代码），Catching the crashing exception（hook exception handler），Leveraging mechanisms provided by the OS（借助os提供的保护机制）。

针对上面提到的两个主要毛病，有许多`有代价`的解决方案。

## 解决方案<br>
### 1. Static Instrumentation<br>
`prerequisites：源码和编译工具链`<br>
在有源码和编译工具链的情况下，在源码里插入检测机制，重新编译.<br
**个人感觉这种办法没啥用，因为嵌入式设备厂商一般不开放源代码,如果是内部人士，或许可行**<br>

### 2. Binary Rewriting<br>
`prerequisites: 固件和设备`<br>
在获取固件的情况下，对firmware的二进制文件插入相应的检测技术，其实难度还挺大的（😀。<br>

### 3.Physical Re-Hosting<br>
`prerequisites: 源码, 编译链, 不同设备`<br>
将源代码重新编译到其他设备上，其他设备可能拥有更高算力。<br>

### 4.Full Emulation<br>
全模拟，把固件的所有行为都模拟一遍。<br>
实际上模拟固件要花费的经力还挺大的。特别是**外设模拟**<br>

### 5.Partial Emulation<br>
退而求其次，只模拟固件，不模拟外设<br>

### 6.Hardware-Supported Instrumentation<br>
利用硬件支持的调试接口(例如，UART)，可以利用高级接口获取更多的运行时信息，辅助fuzz发现crash。<br>

## 题外话<br>
在我读的`What You Corrupt Is Not What You Crash: Challenges in Fuzzing Embedded Devices`论文中，论文作者提出了6种启发式方法，其实对于这些个启发式方法的实现，这些个方法，是基于模拟器修改的，在模拟器的基础上增加了一个额外的监控器，包括`Segment Tracking，Format Specifier Tracking，Heap Object Tracking，Call Stack Tracking，Call Frame Tracking，Stack Object Tracking`<br>
其实感觉吧，也算是Emulation的延申吧。

另外，在读`Discovery and Identification of Memory Corruption Vulnerabilities on Bare-metal Embedded Devices`时，作者通过`binary rewriting`方法插入检测机制，还算比较有趣。<br>
`µSBS: Static Binary Sanitization of Bare-metal Embedded Devices for Fault Observability`跟上面的一篇文章讲得是同一个东西，哈哈哈哈，这种情况还是比较罕见的，但是不得不承认，看完后对于它的实现原理有了更深入的了解。<br>
之后应该会针对这个工具的实现原理进行一个细致的分析，并重写得到一个新的解析器。<br>


