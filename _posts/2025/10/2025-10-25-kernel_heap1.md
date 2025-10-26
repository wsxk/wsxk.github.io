---
layout: post
tags: [kernel_pwn]
title: "kernel heap 1: slab & 内核heap防御机制 & kernel heap常见漏洞类型"
author: wsxk
date: 2025-10-25
comments: true
---

- [写在前面](#写在前面)
- [1. slab allocators](#1-slab-allocators)
  - [1.1 slab逻辑管理结构](#11-slab逻辑管理结构)
  - [1.2 kernel申请内存方法](#12-kernel申请内存方法)
  - [1.3 观测方法](#13-观测方法)
- [2. kernel heap protections](#2-kernel-heap-protections)
  - [2.1 Freelist Randomization](#21-freelist-randomization)
  - [2.2 Hardened Freelist](#22-hardened-freelist)
  - [2.3 Hardened Usercopy](#23-hardened-usercopy)
  - [2.4 kaslr](#24-kaslr)
- [3. 常见的kernel heap利用技巧](#3-常见的kernel-heap利用技巧)
  - [3.1 kernel heap 漏洞](#31-kernel-heap-漏洞)


# 写在前面<br>
slab的相关文章[https://wsxk.github.io/linux_kernel_basic_one/#slab-alloctor](https://wsxk.github.io/linux_kernel_basic_one/#slab-alloctor)<br>
以及[linux内核基础 三 slub分配器详解](https://wsxk.github.io/linux_kernel_basic_three/)其实已经写了一部分，也相当于是故地重游哩~<br>

# 1. slab allocators<br>
内核的内存分配机制和`glibc`不同的主要原因有：<br>
```
1. 性能：内核对于内存分配的性能要求是比较高的，不能用复杂的glibc机制来维护内核内存
2. 效率：glibc管理内存用到的元数据(metadata)太多了，内核不适用。
```

想了解slab的详细原理可以参考我的前2篇文章，我觉得我那2篇文章写的还真不赖，这里不用特地重复写了。<br>
## 1.1 slab逻辑管理结构<br>
总的来说，在`kernel`中，专门维护某个特定大小的内存的结构体叫做`kmem_cache`<br>
```
kmem_cache//专门维护某个大小的内核内存
    kmem_cache_cpu //维护某个slab（通俗的说，是buddy system申请来的pageblock）
        void **freelist;      // 当前 CPU slab 的空闲对象链表//维护当前使用的slab当中的objects
        struct page *page;    // 当前 CPU 正在使用的 slab
        struct page *partial; // 当前 CPU 局部保存的一些“部分填满”的 slab 列表，当前 CPU正在使用的slab满了时，会优先从这里取slab替换

    kmem_cache_node //统一管理多个slab，基于内存控制器的数量的数组（通常来说，一根内存就代表有一个内存控制器）
        struct list_head slabs_partial // 部分使用的slab
        struct list_head slabs_full //已经无剩余容量的slab
        struct list_head slabs_free //完全没用到的slab

    struct list_head list//系统全局的slab_caches链表，所有的kmem_cache都会挂入此链表。
```
一个slab当中的object的名称叫做`slot`，下图中每个`256 bytes`都是一个`slot`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251014000028.png)

## 1.2 kernel申请内存方法
```
void *kmalloc(size_t size, gfp_t flags) 
// 默认情况下会在 通用目的的kmem_cahce中申请堆块，比如 kmalloc-8k
// 如果 GFP_KERNEL_ACCOUNT被设置，此时说明这块内存是不受信任的（通常用在从用户态里获得数据），此时会从 kmalloc-cg-8k 中申请内存
```

## 1.3 观测方法<br>
`cat /proc/slabinfo`非常有用<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251014232119.png)



# 2. kernel heap protections<br>
跟用户态heap一样，内核heap也有很多很多保护机制。<br>
## 2.1 Freelist Randomization<br>
`freelist randomization`指的是在申请一个新slab后，其中的slot是并不是按顺序排序的，而是乱序排序，示意图如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251017201006.png)
<br>

配置了 **CONFIG_SLAB_FREELIST_RANDOMIZATION** 后开启。<br>
在这个配置下`kmem_cache`中会添加一个数组结构，用来存放随机排序的序列。(只确定了刚开始申请到slab中的freelist序列)<br>
```c
#ifdef CONFIG_SLAB_FREELIST_RANDOM
    unsigned int *random_seq;
#endif
```
当然，因为freelist是单向链表（后进先出），黑客在申请多个slot后，能够按照他想要的顺序释放它。<br>

## 2.2 Hardened Freelist<br>
`Hardened Freelist`其实可以类比`tcache中的safelinking机制`,本质上就是freelist中的指针不再单纯指向另一个slot，而是再异或上两个其他值:`ptr ^ s->random ^ &ptr`,这里的&ptr指存放ptr的slot的地址<br>
配置了**CONFIG_SLAB_FREELIST_HARDENED**后开启。<br>
```c
static inline freeptr_t freelist_ptr_encode(const struct kmem_cache *s,
					    void *ptr, unsigned long ptr_addr) {
	unsigned long encoded;
#ifdef CONFIG_SLAB_FREELIST_HARDENED
	encoded = (unsigned long)ptr ^ s->random ^ swab(ptr_addr);
#else
	encoded = (unsigned long)ptr;
#endif
	return (freeptr_t){.v = encoded};
}
```

当然，如果有uaf漏洞或者oob漏洞，有办法泄露其值。<br>

## 2.3 Hardened Usercopy<br>
`Hardened Usercopy`本质上就是限制从用户态传过来的数据能在内核态中存储的空间位置及其大小。<br>
通过配置了`CONFIG_HARDENED_USERCOPY`后开启<br>
配置成功后，在内核中存储用户态值除了传入指针外，还要传入其偏移和大小才行。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251017202510.png)
但是这个机制也有缺点：<br>
这个机制实现在如下函数中:<br>
```c
kmem_cache_create_usercopy
__check_heap_object
```
`copy_from_user`和`copy_to_user`里会运用上述2个函数，但是`memcpy`没有！！！<br>

## 2.4 kaslr<br>
跟`aslr`类似。要想破解它首先需要泄露基址:<br>
如果内核发生了**非致命性错误(oops)**，那么内核会继续运行，并打印出错信息（所有寄存器），可以通过dmesg查看泄露内容<br>
但是如果内核发生了**致命错误（panic）**，那么内核会直接崩溃，这时候就有信息了。<br>

# 3. 常见的kernel heap利用技巧<br>
## 3.1 kernel heap 漏洞<br>
`kernel heap`的漏洞跟用户态heap漏洞如出一辙<br>
```
oob： 堆溢出问题，顾名思义
uaf： use after free
  -泄露freelist
  -破坏metadata（freelist）
  -任意地址读写原语
overlapping allocations： 堆块重叠
```
我们能够利用这些漏洞来创造条件，帮助我们成功提权！但是仅仅识别这些漏洞还是不够的，尝试利用它们非常重要。而利用它们，就需要对linux kernel的机制有充分的了解，并掌握一些利用trick<br>



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>