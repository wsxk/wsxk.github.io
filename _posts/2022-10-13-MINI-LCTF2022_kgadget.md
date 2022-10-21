---
layout: post
tags: [kernel_pwn]
title: "MINI-LCTF2022 kgadget复现"
date: 2022-10-13
author: wsxk
comments: true
---

PS:请观看完[linux内核基础 二](https://wsxk.github.io/linux_kernel_basic_two/)后练习该题<br>

- [题目分析<br>](#题目分析)
- [遇到的坑<br>](#遇到的坑)
  - [1.新的获取vmlinux工具<br>](#1新的获取vmlinux工具)
  - [2.ROPgadget搜索不全<br>](#2ropgadget搜索不全)
  - [3.gdb用法<br>](#3gdb用法)
  - [4. ret、retf、iretq、iret<br>](#4-retretfiretqiret)
  - [5.使用swapgs_restore_regs_and_return_to_usermode<br>](#5使用swapgs_restore_regs_and_return_to_usermode)
- [reference<br>](#reference)

## 题目分析<br>
只有`kgadget_ioctl`有用
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013212902.png)<br>
但是它的代码其实不太对劲，需要我们自己看汇编<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013212950.png)
主要思路是，cmd=114514后，可以通过param（v3 rdx）传入的指针获得函数地址，随后调用<br>
可以看到，它对pt_regs的其他值做了限制，导致你只能使用`r8 r9`两个寄存器<br>
该题目利用了`direct mapping of all physical memory`的漏洞，通过mmap大量的用户态内存地址，写入同样的payload，在物理内存映射区里选择一个地址，有很大的概率会命中rop_chain，最终完成利用。<br>
详细细节可以看[arttnba3的blog](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#%E4%BE%8B%E9%A2%98%EF%BC%9AMINI-LCTF2022-kgadget)

## 遇到的坑<br>
### 1.新的获取vmlinux工具<br>
github搜索 vmlinux-to-elf<br>
可以安装该工具，能够成功提取出可供ida分析的内核文件<br>
```
vmlinux-to-elf input_file output_file
```
### 2.ROPgadget搜索不全<br>
github安装ropper<br>
```
ropper --clear-cache
ropper -f vmlinux --nocolor > gadget.txt
```
### 3.gdb用法<br>
```gdb
gdb vmlinux
add-symbol-file xxx.ko addr -s .data addr -s .bss addr (addr内核root权限通过 cat /sys/module/xxx.ko/sections/.text .data .bss获得)
target remote :1234
b entry_SYSCALL_64 #可以在执行syscall后进行调试
```
### 4. ret、retf、iretq、iret<br>
call 对于 ret（pop rip）<br>
call far 对于 retf（从栈顶弹出 EIP >> CS >> EFLAGS >> ESP >> SS）<br>
iret（4字节） iretq（8字节版本）<br>
### 5.使用swapgs_restore_regs_and_return_to_usermode<br>
因为开启了`kpti`保护

## reference<br>
[arttnba3的blog](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#%E4%BE%8B%E9%A2%98%EF%BC%9AMINI-LCTF2022-kgadget)<br>
[https://blog.csdn.net/qq_50332504/article/details/124145581](https://blog.csdn.net/qq_50332504/article/details/124145581)