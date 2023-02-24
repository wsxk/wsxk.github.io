---
layout: post
tags: [iot]
title: "Address Sanitizer"
date: 2023-2-23
author: wsxk
comments: true
---

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

- [前言](#前言)
- [AddressSanitizer composition](#addresssanitizer-composition)
  - [1. Shadow Memory](#1-shadow-memory)
  - [2. Instrumentation](#2-instrumentation)
  - [3. Debug Allocator](#3-debug-allocator)
- [AddressSanitizer Algorithm](#addresssanitizer-algorithm)
  - [1. shadow memory](#1-shadow-memory-1)
  - [2. Instrumentation](#2-instrumentation-1)
  - [3. Run-time Library](#3-run-time-library)
  - [4. Stack And Globals](#4-stack-and-globals)
  - [5. Thread](#5-thread)
- [实践](#实践)


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
### 1. shadow memory<br>
`AddressSanitizer`用的`shadow memory`映射的规律是这种形式<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224151453.png)

> 1. Addr 表示实际的内存虚拟地址
> 2. Scale 表示伸缩的范围，因为Asan用 1byte 表示 8个byte内存状态，因此Scale 为3
> 3. offset 表示一个偏移，这是基于一个实际意义的考量，因为 shadow memory 的内存位置 不能和原本已使用的内存空间冲突，所以我们需要找到一块内存的位置，保证程序不会使用到它。这就是偏移的用处

关于`offset`的设置，实际上`shadow memory`用到的内存范围为`offset` ~ `offset+max(系统最大内存)/8`。因为需要保证不会占用程序启动时需要的内存区域，offset在`32`位下通常是**Offset = 0x20000000  2^29^** . 而在`64`位下，通常是**Offset = 0x0000100000000000 (2^44^)**<br>
在编译器选项`-fPIE/-pie` (compiler flags on Linux) 下，通常使用`offset=0`的形式（因为0地址区间不会被使用到）<br>

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224152139.png)
值得注意的一点是，因为 `shadow memory` 映射了整个虚拟内存，当然会出现影子内存映射到映射内存的位置，我们称其为`bad`区域，对此的处理是，将`bad`区域设置为不可访问区域。<br>

**关于影子内存表示的值的意义**<br>
> 1. 0 表示 8个字节均可访问
> 2. 1，2...k...7 表示前k个字节均可访问
> 3. 负数表示前 8个字节均不可访问


**实际上scale可以是1~7的任意值，scale越大，shadow memory占据的位置越小(1/2^N^)，然而 使用的 redzong区域会更大(2^N^)


### 2. Instrumentation<br>
如果是进行一个8字节的访问，会插入以下的公式
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224155138.png)
如果是 1，2，4字节访问，会使用以下公式
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224155210.png)

插桩设置会放置在llvm optimization后（因为优化后的内存访问会减少，减少插桩的数量，提高程序运行效率）<br>
因为插桩默认是 访问几个字节就几字节对齐，所以会导致有些越界访问不会被检查到。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224155555.png)
像这种风格的代码，*u=(int *)((char * )a+6)时，不会发生触发crash。因为在检测时，他是符合规则 `*ShadowAddr=0` 的<br>
值得一提的是，虽然有漏报，但是没有误报情况

### 3. Run-time Library<br>
这个东西其实是用来管理 `shadow memory`的(比如初始化)，并且对`malloc`函数，`free`函数进行一些hook操作，使其在申请/释放前，先对`shadow memory`的相应位置进行值的更改。<br>
对于`malloc`，会申请更大的内存，存放`redzone`。<br>
对于内存分配器`allocator`的操作，以下这个解释比较形象<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224160602.png)
对于`free`，并不会立即释放这个内存，而且把它挂在一个空链表中，被称作`quarantine`，即隔离。这个空链表遵循`FIFO`规则，因此，当空链表已经满了后，发生的 `use-after-free`就不会什被check到。<br>


### 4. Stack And Globals<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224161052.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224161145.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224161159.png)

### 5. Thread<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224161352.png)


## 实践<br>
`AddressSanitizer`已经被集成到了`llvm`上，下载`llvm`后即可使用。<br>
使用文档如下<br>
[https://github.com/google/sanitizers/wiki/AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)<br>
实际跑个代码试试
```c
#include <stdlib.h>
int main() {
  char *x = (char*)malloc(10 * sizeof(char*));
  free(x);
  return x[5];
}
```
```
clang -fsanitize=address test.c -o test_asan //编译一个有asan的
clang test.c -o test //原始的
```

运行 有asan的会出现如下内容:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224161809.png)

ida查看<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224161833.png)
再来看看原版的
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230224161854.png)
