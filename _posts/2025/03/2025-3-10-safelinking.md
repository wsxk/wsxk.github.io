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
  - [2.1](#21)


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
## 2.1 

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


