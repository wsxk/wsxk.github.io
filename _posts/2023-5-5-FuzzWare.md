---
layout: post
tags: [iot]
title: "Fuzzware: Using Precise MMIO Modeling for Effective Firmware Fuzzing"
author: wsxk
date: 2023-5-5
comments: true
---

- [写在前面](#写在前面)
- [技术背景](#技术背景)
  - [1. 目标固件](#1-目标固件)
  - [2. Memory-mapped IO(MMIO)](#2-memory-mapped-iommio)
  - [3. Interrupts and DMA](#3-interrupts-and-dma)
  - [4. Re-Hosting Embedded Systems](#4-re-hosting-embedded-systems)
- [有关MMIO的模拟的问题讨论](#有关mmio的模拟的问题讨论)
  - [1. Input Overhead](#1-input-overhead)
  - [2. 当前的MMIO models 方法](#2-当前的mmio-models-方法)


## 写在前面<br>
这边笔记是对于最近看的一篇论文`Fuzzware: Using Precise MMIO Modeling for Effective Firmware Fuzzing`的总结。<br>
**这是一个结合了符号执行技术（angr）和模糊测试技术（fuzz）以及仿真技术（Unicorn）的产物，目的是更有效的模拟固件执行，发现更多的漏洞**<br>

## 技术背景<br>
在看这篇论文前，需要了解一些前置知识。<br>
### 1. 目标固件<br>
这篇论文关注的是`monolithic firmware`，即[https://wsxk.github.io/iot%E5%85%A5%E9%97%A8%E4%B8%80/](https://wsxk.github.io/iot%E5%85%A5%E9%97%A8%E4%B8%80/)提到的III型嵌入式设备，没有系统存在。<br>

### 2. Memory-mapped IO(MMIO)<br>
同样可以看一下[https://wsxk.github.io/iot%E5%85%A5%E9%97%A8%E4%B8%80/#iot%E8%AE%BE%E5%A4%87%E7%BB%84%E6%88%90](https://wsxk.github.io/iot%E5%85%A5%E9%97%A8%E4%B8%80/#iot%E8%AE%BE%E5%A4%87%E7%BB%84%E6%88%90)提到的，MMIO是嵌入式设备cpu和外设交互的一种方式（顺道一提，DMA其实是基于MMIO实现的）<br>
通过直接将外设的寄存器或存储区域直接映射到内存中的一个位置，使得CPU可以通过LOAD/STORE指令直接访问数据。<br>
这是一个很高效的方法。<br>

### 3. Interrupts and DMA<br>
中断也是一种cpu和外设交互的方式，当外设的内容到达时，硬件可以会触发中断，通知cpu立刻处理到达的数据。<br>
DMA是通过MMIO实现的方法，外设拥有权限直接往内存写入东西，不需要通知CPU。(本质上是有一个DMA控制器来管理外设和内存的数据传输，传输完成后会发出中断给cpu，让cpu执行需要的操作)<br>

### 4. Re-Hosting Embedded Systems<br>
本质是是把嵌入式固件，从嵌入式设备（物理设备）上移动到PC机上，利用PC机的强大资源进行模糊测试。然而模拟固件运行也有许多难题：模拟CPU指令的执行是很容易做到的，但是模拟外设与CPU交互这一方面存在巨大困难。<br>
嵌入式固件中 cpu和外设交互的方式主要有3种：MMIO, Interrupts, DMA（PMIO也有，但是不是很常见）<br>
模拟的主要问题处在对这3种方式的实现上。<br>
这篇论文主要关注的是`MMIO`的实现。<br>
目前，对`MMIO`模拟的解决方案有2种。<br>

> 1. 以qemu为首的重写派: qemu会针对每个MMIO的外设，完全的复现在嵌入式设备上的行为。这种办法在用户体验上来说是很好的，对于qemu开发人员来说还是很累的：他们需要针对每个外设，查看硬件手册。<br>
>
> 2. approximation派： 引入fuzzer来解决MMIO的访问问题。本质上来说，fuzzer生成的随机值是可以作为硬件生成的返回值的。这个方法很好，因为不需要关系外设的实现细节和MMIO的前置知识。然而，要想实现这个技术也有困难：fuzzer需要给每个MMIO外设提供随机输入（这通常有很多个），而大多数的外设提供的值是没有意义的，也就是说，fuzzer很多时候再做无用功，并不能很好的适应现实的嵌入式设备。


## 有关MMIO的模拟的问题讨论<br>
这一节讨论 为什么用fuzzer解决MMIO问题会导致很大的开销，另外探寻一下现存的去除这些开销的方法。<br>

### 1. Input Overhead<br>
假设一种天真的方法，其中模糊器生成的随机字节流中的比特被用作硬件生成的值（即，从固件的角度来看，由硬件MMIO寄存器提供的值）。我们将这些比特称为模糊器变异输入空间，然后由固件逻辑进行处理。这个输入空间既包含相关的位，即影响固件逻辑的位，也包含输入开销。对于每个MMIO访问，我们区分两种类型的输入开销。<br>

> 1. Full input overhead : 即fuzzer生成的随机数据，没有一个bit对固件运行是相关的，事实上，这可以用一个任意值替代，不需要fuzzer生成的随机数据。
>
> 2. Partial input overhead : 即，fuzzer生成的随机数据中，只有部分bits是有用的。比如访问一个4字节的内存，然而其中只有1字节是需要用到的。剩下的3字节是没有用的

对于fuzzer来说，有一部分值没有被使用到可以说是十分地浪费资源（因为每一个随机输入都是fuzzer通过变异机制得到的，都需要耗费一定的资源）。<br>
论文作者提出一个例子帮助我们更好的理解。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230505220203.png)
这个函数的功能很简单，首先在轮询，等待有数据进入，需要处理；<br>
然后触发GPIO的写操作（可能是开个led灯，anyway）**注意，GPIO也利用了MMIO机制**<br>
最后取了数据，返回这个数据的第一个字节。<br>

在步骤1，对于fuzzer来说，HAS_DATA是一个4字节的值，通过随机值喂入，让他触发返回数据的逻辑是相对困难的，因为只有一个特定的值可以被接受。这是一个很典型的`full input overhead`的例子。<br>

在步骤2，虽然向GPIO写入可能被视为MMIO写入操作，但GPIO位通常被打包到寄存器中，32位代表32个GPIO引脚。因此，要在不影响附近位的情况下执行GPIO操作，我们必须读取4个字节，翻转所需位，然后将结果写回。一开始读取的初始化的4字节是没有用的（也是`full input overhead`）。<br>

在步骤3，你读取的4个字节中，有3个字节其实是固件运行不需要的（`partial input overhead`），这么算下来，咱们发现，其实fuzzer生成的随机数据，基本上都不需要用到，那生成这些数据就没有意义，耗费了时光了。<br>

咱们再看一下另一个例子<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230505221607.png)
这个例子中，有一个`mmio->op`操作，我们需要喂入4字节的随机数据，然而只有4个选择（可以用2bits表示，这造成了`partial input overhead`）。另外，还有一个`mmio->status`操作，也需要4个字节，然而，只需要1bits就能判断这个状态。显然，这也是一个`partial input overhead`。而且，fuzzzer随机喂入的值能达到目标又要花费巨大时间。<br>

### 2. 当前的MMIO models 方法<br>
