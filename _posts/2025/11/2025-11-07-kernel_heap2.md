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
  - [4.1 Heap Spraying —— anit-freelist\_randomization](#41-heap-spraying--anit-freelist_randomization)
    - [4.1.1 UAF破解freelist\_randomization](#411-uaf破解freelist_randomization)
  - [4.3 申请Desirable Objects](#43-申请desirable-objects)



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

## 4.1 Heap Spraying —— anit-freelist_randomization<br>
heap spraying 是一个常见的内核堆利用技术，中文名堆喷射。**在kernel heap当中，堆的布局往往是不可知的，最主要的原因是开启了`freelist_randomization`选项（默认开启）**<br>
比如下图是一个slab中常见的object布局:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251028222255.png)
对于攻击者而言，无法确认下一个分配到的object位于哪个位置。（除非你的堆溢出能够溢出非常多，且覆盖正常object不会对kernel运行造成影响）。<br>
此时可以考虑利用`heap spraying`技术，我们可以申请多个object，让kernel耗尽当前kmem_cache_cpu中的slab，并让其重新申请一个slab：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251028223704.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251028223729.png)
此时再申请受害object后，堆溢出就容易许多了。<br>

应用场景:<br>
```
你有一个堆溢出读 / 写，但是堆布局对你而言是不可知的（比如说开启了 SLAB_FREELIST_RANDOM（默认开启）），你可以预先喷射大量特定结构体，从而保证对其中某个结构体的溢出。
```
### 4.1.1 UAF破解freelist_randomization<br>
前提：关闭kaslr，通过利用uaf漏洞，执行一次任意地址函数调用，且rdi寄存器是一个指针（不可改变），指向的内存区域可控。<br>
办法:执行`commit_creds(rdi)`。rdi指向的内存区域，抄袭init_cred的内容<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251029222456.png)


## 4.3 申请Desirable Objects<br>
在kernel heap场景当中，堆布局是非常困难的。`kmalloc`函数会从 `通用的kmalloc_kmem_cache`中返回对象。然而：**通用cache可以保存许多大小相似的不同对象类型**<br>



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>