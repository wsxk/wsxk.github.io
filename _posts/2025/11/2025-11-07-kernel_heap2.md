---
layout: post
tags: [kernel_pwn]
title: "kernel heap 2: 内核堆利用技巧"
author: wsxk
date: 2025-11-07
comments: true
---

- [1. kernel heap使用范式](#1-kernel-heap使用范式)
- [2.  泄露内核地址篇](#2--泄露内核地址篇)
  - [2.1 Oops泄露kernel地址绕过kaslr](#21-oops泄露kernel地址绕过kaslr)
- [3. 内核堆漏洞](#3-内核堆漏洞)
  - [3.1 oob(out of boundry)：堆溢出](#31-oobout-of-boundry堆溢出)
  - [3.2 UAF](#32-uaf)
  - [3.3 overlapping allocation](#33-overlapping-allocation)
- [4. 内核堆利用技巧](#4-内核堆利用技巧)
  - [4.1 Heap Spraying](#41-heap-spraying)



# 1. kernel heap使用范式<br>
```c
cachep = (kmem_cache *)kmem_cache_create("kheap_obj", 472LL, 0LL, 84156416LL, 0LL);//大小为472
v6 = kmem_cache_alloc(cachep, 0x400CC0LL);//从这个kmem_cache中分配一个slot
kmem_cache_free(cachep, filp->private_data);//释放右侧chunk，归入cahchep中。
kmem_cache_destroy(cachep); //摧毁kmem_cache
```

# 2.  泄露内核地址篇<br>
## 2.1 Oops泄露kernel地址绕过kaslr<br>
触发Oops后的情景如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251023001013.png)
`R10的值ffffffff82a58c20为内核地址段，R12的值0xffff8880043a7000为物理映射区域，但是实际上，它会指向kernel的heap基址！`<br>

# 3. 内核堆漏洞<br>
通常，内核堆漏洞分为如下三种：<br>
## 3.1 oob(out of boundry)：堆溢出<br>
顾名思义，其实就是在一个slot中填充多于其大小的内容，覆盖下一个slot中的其他值。<br>
只有这个漏洞通常能够泄露下一个slot中的信息（如果有机密信息的话）<br>

## 3.2 UAF<br>
uaf无需多说了。

## 3.3 overlapping allocation<br>
同理。<br>

# 4. 内核堆利用技巧<br>
通常情况，找到了漏洞是以远远不够的。如何利用漏洞达成目的才是重点。为了达成目的，我们需要学习漏洞的利用手段.<br>

## 4.1 Heap Spraying<br>
heap spraying 是一个常见的内核堆利用技术，中文名堆喷射。<br>




<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>