---
layout: post
tags: [iot]
title: "retrowrite"
date: 2023-2-27
author: wsxk
comments: true
---

- [前言](#前言)
- [overview](#overview)
  - [1. Processing](#1-processing)
  - [2. Symbolization](#2-symbolization)
  - [3. Instrumentation Passes](#3-instrumentation-passes)
  - [4. Instrumentation Optimization](#4-instrumentation-optimization)
  - [5. Reassembly(ASM)](#5-reassemblyasm)
  - [思路优点](#思路优点)
- [框架实现方法](#框架实现方法)


## 前言<br>
这篇文章其实是记录论文`RetroWrite: Statically Instrumenting COTS Binaries for Fuzzing and Sanitization`的内容（真·顶会论文），very牛逼<br>
论文作者提出了一个开源框架`retrowrite`,在[https://github.com/HexHive/retrowrite](https://github.com/HexHive/retrowrite)可以找到他的代码<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230227201328.png)
可以看到有500+star，60+fork，起码还是有不少人尝试使用过了（吐槽某个神秘uSBS<br>
在这篇论文中，作者首先对当前的二进制重写工具进行了一系列评估，并认为它们的效率太差了。于是提出了一个新的二进制重写思路，并使用这个思路写出了新的开源二进制框架，并表示：`我的开源框架很猛，效率比当前的其他框架，效率能加个好几倍！`<br>


## overview<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230227202037.png)
这个工具的工作流程被总结成了5步。<br>
即`1.Processing 2.Symbolization 3.Instrumentation Passes 4.Instrumentation Optimization 5.Reassembly`<br>

### 1. Processing<br>
在这个阶段，该框架做了如下几件事:
**分析二进制文件并加载 text & date section，还从中抽取出符号信息和重定位信息。**<br>
**将二进制码用 `linear sweep`方法反汇编成汇编代码**<br>
**在前两步的基础上，进行轻量的分析，构造cfg（直接跳转才能被加边）**<br>

### 2. Symbolization<br>
利用 第一步生成的 cfg 以及 重定位信息 识别在text、data段中的可符号化的常数，并用`assembly labels`来替换他们。<br>
此时已经有了带有`assembly labels`的汇编代码(assembly)<br>
再具体而言，符号化的操作经过了3个步骤<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230227204425.png)
**首先找出 call和jump的指令，并用assembly labels替换它们**<br>
**其次找出 PIC中找到计算PC-relative addresses的指，用assembly labels替换它们**<br>
**最后在data段中，根据重定向条目指向的偏移，用assembly labels替换它们**<br>


### 3. Instrumentation Passes<br>
就是进行一些插桩操作<br>

### 4. Instrumentation Optimization<br>
对现有的插桩进行优化，包括分析插桩并确定需要用到的寄存器数量以及副作用<br>
这一步归结起来主要做2件事<br>
**选择性的保留/恢复 状态变化**<br>
**为插桩来申请寄存器**<br>

### 5. Reassembly(ASM)<br>
再重新将汇编代码转变成二进制代码<br>

### 思路优点<br>
一、不需要将汇编代码再提升一个层次到IR中,（通常这种做法很耗时，而且容易出错<br>
二、轻量，而且可以直接在 现有的反汇编器生成的反汇编代码中 进行操作。<br>
**三、利用relocation、PC-relative addressing以及恢复的控制流，来确定代码中的指针信息**<br>
四、因为不是使用启发式算法，因此没用误报和漏报，说明这个框架是通用的，应用范围很广。<br>


## 框架实现方法<br>
