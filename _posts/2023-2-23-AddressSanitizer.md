---
layout: post
tags: [iot]
title: "Address Sanitizer"
date: 2023-2-23
author: wsxk
comments: true
---

- [前言](#前言)
- [AddressSanitizer composition](#addresssanitizer-composition)
  - [1. Shadow Memory](#1-shadow-memory)
  - [2. Instrumentation](#2-instrumentation)
  - [3. Debug Allocator](#3-debug-allocator)
- [AddressSanitizer Algorithm](#addresssanitizer-algorithm)


## 前言<br>
这篇内容其实是来自于文章**AddressSanitizer: A Fast Address Sanity Checker** 的阅读笔记。<br>
其实它可能跟`iot`关系不是很大，但是他和我做得iot相关的毕设关系很大(😀。<br>
我的毕设似乎是要往arm架构上迁移x86的一些安全检测技术(比如AddressSanitizer，它是现在普遍应用与软件安全检测的一种技术）。<br>
然而，AddressSanitizer到底是什么个东西，我也不是很懂。<br>
所以看一下这篇论文还是很重要的.<br>

## AddressSanitizer composition<br>
`AddressSanitizer`，简称`asan`,构建起这个技术的组成有3种。即**shadow memory（影子内存），插桩（instrumentation），检测分配器（debug allocator）** <br>

### 1. Shadow Memory<br>
`shadow memory`，其实是献祭部分内存，表示其他内存状态的一种方法。<br>
比如，用位于0x4000的内存的值，用来表示0x8000位置的内存的状态。<br>

### 2. Instrumentation<br>
`插桩`，直白地说，就说往源码或二进制代码里插入一些检测方法，帮助用户截获程序异常信息。<br>

### 3. Debug Allocator<br>
要想实现这个技术，需要有一个专门为其定制的`Allocator`,因为要做一些heap检测时，需要hook类似于`malloc`和`free`函数，使得其能够多分配一些内存存储`redzone`，释放时不会立即释放。<br>


## AddressSanitizer Algorithm<br>
