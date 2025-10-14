---
layout: post
tags: [kernel_pwn]
title: "kernel heap 1: slab"
author: wsxk
date: 2025-10-25
comments: true
---

- [写在前面](#写在前面)
- [1. slab allocators](#1-slab-allocators)
  - [1.1 slab逻辑管理结构](#11-slab逻辑管理结构)
  - [1.2 开发及观测方法](#12-开发及观测方法)
- [2. kernel heap protections](#2-kernel-heap-protections)
- [3. exploit the kernel](#3-exploit-the-kernel)


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

## 1.2 开发及观测方法<br>
这章节主要讲实战部分<br>



# 2. kernel heap protections<br>


# 3. exploit the kernel<br>



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>