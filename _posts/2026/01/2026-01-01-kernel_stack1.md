---
layout: post
tags: [kernel_pwn]
title: "kernel stack 1: 环境搭建& kernel stack保护机制"
author: wsxk
date: 2026-01-01
comments: true
---

- [1. 写在前面](#1-写在前面)
- [2. 环境介绍](#2-环境介绍)
  - [2.1 bzImage](#21-bzimage)
  - [2.2 文件系统：initramfs.cpio.gz](#22-文件系统initramfscpiogz)
  - [2.3 run.sh](#23-runsh)
- [3. linux kernel 保护机制](#3-linux-kernel-保护机制)


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
## 2.1 bzImage<br>
对于bzImage格式的文件，我们可以通过
[extract-image.sh](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/extract-image.sh)提取`vmlinux`，这实际上是内核可执行文件。<br>
```
./extract-image.sh bzImage > vmlinux
```
提取后，可以用`ROPgadget --binary vmlinux > gadget.txt`来提取gadgets.<br>

## 2.2 文件系统：initramfs.cpio.gz<br>
可通过[decompress.sh](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/decompress.sh)进行解压<br>
```
./decompress.sh 
```
解压后可以看到里面的目录<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251129225349.png)
通常`etc`目录里有文件的启动脚本，我们可以进去修改一下用户权限，来获得`root`，获得`root`的原因是方便你调试。当然，实际执行exp时还是得切换回普通用户。<br>
写好利用脚本后，我们通常需要把脚本编译好后打包进文件系统里.<br>
[compress.sh](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/compress.sh)
```
./compress.sh 
```

## 2.3 run.sh<br>
实际上就是`qemu`的启动命令<br>
```
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,+smep,+smap \
    -kernel vmlinuz \
    -initrd initramfs.cpio.gz \
    -hdb flag.txt \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -no-reboot \
    -append "console=ttyS0 kaslr kpti=1 quiet panic=1"
    -s
```
其中：<br>
> -m 128M :设定虚拟机内存，通常情况下虚拟机起不来就会调大内存
> -cpu ：设置cpu模式，可以添加 smep/smap保护机制
> -kernel ： 选择kernel
> -initrd ：选择文件系统
> -append : 添加其他启动项，比如 kaslr，或者 kpti保护机制
> -s : 开启远程调试接口，可通过 gdb启动后 target remote 127.0.0.1:1234进行调试

其他选项可以参考[https://manpages.debian.org/jessie/qemu-system-x86/qemu-system-x86_64.1.en.html](https://manpages.debian.org/jessie/qemu-system-x86/qemu-system-x86_64.1.en.html)<br>

# 3. linux kernel 保护机制<br>

```
1. Kernel stack cookies (or canaries)： 同用户态canary，内核编译时就确定了，不能通过其他方式取消

2. Kernel address space layout randomization (KASLR)： 同用户态aslr，能通过-append中 添加 nokaslr关闭

3. Supervisor mode execution protection (SMEP)： 进入内核态后，将页表中所有用户态相关的页设置为不可执行。通过标记CR4寄存器的第20个bit来使能。 -cpu选项添加 +smep可开启， -append 添加 nosmep关闭

4. Supervisor Mode Access Prevention (SMAP) ：进入内核态后，将页表中所有用户态相关的页设置为不可读写。通过标记CR4寄存器的第21个bit来使能。 -cpu选项添加 +smap可开启， -append 添加 nosmap关闭

5.Kernel page-table isolation (KPTI)： 这个功能启动后，内核会将用户页表和内核页表完全分离。内核页表保留了用户态和内核态的地址空间，但只在内核态使用。 用户页表只保留用户态地址和最少的内核空间地址。 -append中添加 kpti=1开启， nopti关闭。
```



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>