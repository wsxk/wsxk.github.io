---
layout: post
tags: [kernel_pwn]
title: "kernel stack 1: 环境搭建& kernel stack保护机制"
author: wsxk
date: 2025-12-21
comments: true
---

- [1. 写在前面](#1-写在前面)
- [2. 环境介绍](#2-环境介绍)
  - [2.1](#21)


# 1. 写在前面<br>
yysy，先看`kernel heap`再看`kernel stack`，我也是真挺抽象的哈哈<br>
之前复现过几篇，欢迎取用：<br>
[linux内核基础 一 常识](https://wsxk.github.io/linux_kernel_basic_one/)<br>
[qwb2018 core 复现 ROP](https://wsxk.github.io/qwb2018_core/)<br>
至于为什么要再写，是因为从现在的角度回望过去，觉得自己写的还是太潦草了，并没有真的理解利用的方法。准备洗心革面，从头再来<br>

# 2. 环境介绍<br>
以CTF的常规kernel题目出发，文件附录里通常会有以下几个内容:<br>
```
bzImage: 也成为vmlinuz文件，是vmlinux的压缩格式，可通过脚本提取出vmlinux（方便调试）
initramfs.cpio.gz: linux文件系统，由cpio压缩后再由gzip压缩而成，里面保护linux系统的目录
run.sh： 用qemu运行内核的脚本
```
## 2.1 


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>