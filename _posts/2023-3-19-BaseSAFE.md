---
layout: post
tags: [iot]
title: "BaseSAFE: Baseband SAnitized Fuzzing through Emulation"
author: wsxk
date: 2023-3-19
comments: true
---

- [前言](#前言)
- [简介](#简介)
  - [1. fuzz](#1-fuzz)
  - [2. unicorn](#2-unicorn)
  - [3. Cellular Baseband](#3-cellular-baseband)
- [程序运行逻辑](#程序运行逻辑)
- [安全检测实现思路](#安全检测实现思路)


## 前言<br>
这是取自 `BaseSAFE: Baseband SAnitized Fuzzing through Emulation` 论文的笔记<br>

## 简介<br>
这篇论文将 `fuzz`技术和 `unicorn`技术结合起来，针对 `cellunar baseband`进行安全测试。<br>

### 1. fuzz<br>
不用多说了，大家肯定都耳熟能详<br>

### 2. unicorn<br>
来自`qemu`的分支，github网址在[https://github.com/unicorn-engine/unicorn/tree/7e4754ad008f0dac6d990a7a997a764062b35d04](https://github.com/unicorn-engine/unicorn/tree/7e4754ad008f0dac6d990a7a997a764062b35d04)<br>
`unicorn`从`qemu`出家，当然支持许多架构的模拟运行，于`qemu`不同的是，`unicorn`提供了更方面用户使用的`API接口`，**unicorn暴露了读写内存的函数，以及hook特定地址跳转到回调函数的功能，然而作者还对第三方unicorn-rs bindings进行了拓展**<br>
Unicorn的工作原理和qemu类似<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319141716.png)

### 3. Cellular Baseband<br>
手机除了普通的应用处理器（跑程序）外，还有一种`baseband processor`，<br>
这种处理器用于处理信号的（比如5G信号传来，翻译成手机信息）<br>

## 程序运行逻辑<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319151047.png)
相关API可以去文章里看<br>


## 安全检测实现思路<br>
利用了`unicorn`提供的API以及自扩展的`rs-bindings`，对内存分配进行hook，执行自己的策略<br>

在arm32嵌入式设备中，`memory corruption`和`corrupted memory use`通常是分离的，在这是，`fuzzer`通常开始了下一轮迭代了（重新起一个新的程序），这时会去除所有的bug的信息。<br>
为了解决这个问题，作者插入了一个`drop-in allocator`<br>
> 1. 用 unicorn提供的hooking functionality，直接在JITed code里插入条件检查和回调函数
> 2. 事先分配一个很大的内存，访问时，触发access-hook，分配时会使用自己的分配器，取消access hook，根据分配大小，每个被分配块两边都会挂上一个canary块
> 3. 取消分配时，又挂上access hook，chunksize置为0（独立于结构体之外,因此不会被消除）
