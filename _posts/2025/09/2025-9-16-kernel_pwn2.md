---
layout: post
tags: [kernel_pwn]
title: "kernel security 2: ptracticing"
author: wsxk
date: 2025-9-16
comments: true
---

- [1. kernel 环境搭建](#1-kernel-环境搭建)
  - [1.1 自安装](#11-自安装)
  - [1.2 一键式脚本](#12-一键式脚本)


# 1. kernel 环境搭建<br>
kernel环境搭建需要4个部分:<br>
```
1. 编译器compiler： gcc，用于编译内核模块和内核
2. 内核 kernel： 无需多言，内核elf文件
3. 文件系统 filesystem： 用于存储，有了它，内核才可以存放各种文件。
4. 模拟器 emulator： 多数情况下指的是qemu，用于模拟执行内核
```
## 1.1 自安装<br>
可参考我22年左右写的blog，[https://wsxk.github.io/ubuntu_kernel%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/](https://wsxk.github.io/ubuntu_kernel%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)<br>
自己安装是非常复杂的操作.<br>

## 1.2 一键式脚本<br>
pwn.college提供了一键式脚本:<br>
[https://github.com/pwncollege/pwnkernel/tree/main](https://github.com/pwncollege/pwnkernel/tree/main)<br>
直接运行即可，方便快捷~<br>
考虑到仓库的更新时间，使用ubuntu22虚拟机会是个比较好的选择。<br>



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>