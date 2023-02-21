---
layout: post
tags: [iot]
title: "uSBS arm instrumentation"
date: 2023-2-19
author: wsxk
comments: true
---

这篇文章是基于论文`Discovery and Identification of Memory Corruption Vulnerabilities on Bare-metal Embedded Devices`和`µSBS: Static Binary Sanitization of Bare-metal Embedded Devices for Fault Observability`的理解得到的，有需要的可以自行搜寻(十分好找)<br>
`PS: 这两篇写的工具是一样的，所以结合两篇文章可以更好的理解它的实现原理`<br>

- [uSBS安装](#usbs安装)
  - [step1: 安装ubuntu18](#step1-安装ubuntu18)
  - [step2: 安装docker](#step2-安装docker)
  - [step3: 下载源码](#step3-下载源码)
- [uSBS原理](#usbs原理)
  - [1. Static Disassembler](#1-static-disassembler)
  - [2. Binary Instrumentor](#2-binary-instrumentor)
  - [3. Reassembler](#3-reassembler)
- [实现细节](#实现细节)


## uSBS安装<br>
### step1: 安装ubuntu18<br>
可以参考[我的这篇文章](https://wsxk.github.io/ubuntu_%E5%AE%89%E8%A3%85%E5%B8%B8%E8%A7%84%E6%93%8D%E4%BD%9C/)<br>
**值得注意的是，ubuntu18 清华的镜像是有问题的，很多玄学问题，这里建议去ubuntu官网下载18镜像**<br>

### step2: 安装docker<br>
参考[我的另一篇文章](https://wsxk.github.io/docker_install/)<br>

### step3: 下载源码<br>
[https://github.com/pwnforce/uSBS](https://github.com/pwnforce/uSBS)是他的开源代码<br>
下载后，`make run` 即可<br>

## uSBS原理<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230219191615.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230219191508.png)
上面2张图截取自两篇论文，其实就分了三个步骤(😀<br>
大白话地说，其实就是，**反汇编-插桩-汇编**<br>

### 1. Static Disassembler<br>
反汇编，用的是普罗大众的`capstone`，没什么好说的<br>

### 2. Binary Instrumentor<br>
主要细节在arm的二进制插桩的实现上面。<br>
在实现二进制插桩上，有3个主要的问题<br>
**一、recognizing static addresses**<br>
**二、relocating static addresses after instrumentation**<br>
**三、determining dynamically referenced memory addresses**<br>

> 第一个问题其实是，对于立即数的标量类型和指针 以及 更新指针到新的地址 其实是没有语义上的差别的。
> 第二个问题是，插桩后会导致函数的位置发生了偏移
> 第三个问题是，有些跳转并不是直接跳转，而是间接跳转，比如要跳转的地址= r1*4+r2+r3，这种识别是很困难的

**解决方案**<br>
为了解决上面的问题，文章作者首先将所有的指针引用分成了4种类型<br>
**Code to Code(C2C),Code to Date(C2D),Data to Code(D2C),Data to Data(D2D)**<br>
对于实际上的**C2D and D2D references**，`因为作者不会修改data段的地址，data段中的数据的偏移不会发生改变，所以不需要关注`。<br>
需要关注的是**C2C and D2C references**，`因为插桩会导致代码段中代码的地址偏移发生变化，因此需要修改`,这里作者添加了一个新的段，使用了一个自设计的算法，如下图所示<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230220150807.png)
通过扫描每个指令，更新静态跳转指令的地址，从而实现新修改。<br>
对于间接跳转指令，作者认为，在运行时可以确定它的真实跳转地址，于是乎，他往新插入的代码段中添加了一个mapping表，其中是 原跳转地址——新跳转地址相对于新代码段的基址的偏移。并且插入了一个`mapping routine`函数，发生间接跳转，统一跳到`mapping routine`，`mapping routine`根据`mapping table`中的值，计算新的地址，并跳到那个新地址。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230220151811.png)

解决了二进制插桩产生的问题后，接下来进入`Sanitization Specification`环节，即确定插入的东西是什么<br>
作者原话:<br>
**µSBS utilizes a metadata store that keeps the status of allocated memory bytes.µSBS surrounds every memory value with a so-called redzone representing out-of-bounds memory and marks it as invalid memory in the metadata store.Then, µSBS instruments every memory instruction (i.e., load and store) in order to consult the metadata store whenever the firmware attempts to access memory**
<br>
他用了一个叫做`redzone`的方法来实现，即在每个内存的值上下都环绕一个`redzone`，然后用当程序访问了`redzone`时，会修改`metadata`，实现监控内存访问问题。<br>
***具体而言，其实论文作者对所有的ldr和str指令进行了插桩,其实是插入了检测指令，在执行这2条指令前，先根据要访问的地址咨询一下metadata，看看是否产生了越界***<br>

### 3. Reassembler<br>
重新汇编，用的是十分流行的`keystone`，这也没什么好说的<br>

## 实现细节<br>
**在重新修改函数的跳转时，可能跳转的函数地址太大了，超过原来汇编指令的容量，对此论文作者在修改前，对每个跳转的汇编代码进行替换，换成可以容纳更大偏移的地址的汇编指令。**<br>
然后是`间接跳转`和`间接调用`的处理方式<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE_20230220_192629.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE_20230220_192633.png)

另一个是插桩方式<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230220193424.png)
