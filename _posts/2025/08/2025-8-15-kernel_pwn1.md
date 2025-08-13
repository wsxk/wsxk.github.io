---
layout: post
tags: [kernel_pwn]
title: "kernel security"
author: wsxk
date: 2025-8-15
comments: true
---


- [0. 写在前面](#0-写在前面)
- [1. 内核简介](#1-内核简介)
  - [1.1 什么是操作系统内核](#11-什么是操作系统内核)
  - [1.2 内核专享的外部资源](#12-内核专享的外部资源)


PS:`kernel`，我又回来啦<br>

# 0. 写在前面<br>
其实大三有一段时间是在学习`kernel pwn`的，后来很长一段时间不用又忘记了，如今重拾`kernel`，希望能更系统的学习。<br>
大三的学习记录在[https://wsxk.github.io/linux_kernel_basic_one/](https://wsxk.github.io/linux_kernel_basic_one/)<br>
事到如今，不过是再学一遍罢了~<br>

# 1. 内核简介<br>
## 1.1 什么是操作系统内核<br>
***操作系统（Operation System）*** 本质上也是一种软件，可以看作是普通应用程式与硬件之间的一层中间层，其主要作用便是调度系统资源、控制IO设备、操作网络与文件系统等，并为上层应用提供便捷、抽象的应用接口而` 内核（kernel） `是操作系统最重要的一部分。<br>

## 1.2 内核专享的外部资源<br>
内核专享的外部资源(`external resources`),指只有内核才能访问的资源，即用户态无法直接访问其资源。<br>
这里引出用户态和内核态的概念：用户态就是我们的进程执行的代码空间，内核态指内核执行的代码空间。<br>
也就是说，用户态的程序想要访问一些外部资源，需要先`陷进`内核态，通常这个动作是通过系统调用完成（还有其他，接下来再举例）<br>
一些常见的外部资源如下:<br>
```
1. hlt指令：只在内核态才允许执行该指令
其作用是让 CPU 进入 halt（空闲/省电）状态，停止取指与执行，直到被“唤醒事件”打断。常用于内核的 idle 循环。（idle即空闲状态）

2. in 和 out指令：用于访问I/O端口空间的设备寄存器。其实就是和硬件外设交互，常用于老式设备
（现代的pcie将设备寄存器暴露为某个物理/虚拟地址范围，用普通 mov 读写（再配合内存屏障））


3. 一些特殊寄存器：
cr3:(control register 3) ,它是指向page table的指针，用于虚拟地址和物理地址之间的转换，通常用mov指令就可以修改值，但是必须要是内核态。
MSR_LSTAR (Model-Specific Register, Long Syscall Target Address Register)： 它定义了syscall指令应该跳往哪个函数。 通常用wrmsr和rdmsr来读写这个寄存器
```


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>