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
因为MMIO是有关