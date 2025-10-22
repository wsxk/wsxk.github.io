---
layout: post
tags: [kernel_pwn]
title: "kernel heap 2: practicing"
author: wsxk
date: 2025-11-07
comments: true
---

- [1. kernel heap使用范式](#1-kernel-heap使用范式)
- [2.  Oops泄露kernel地址](#2--oops泄露kernel地址)



# 1. kernel heap使用范式<br>
```c
cachep = (kmem_cache *)kmem_cache_create("kheap_obj", 472LL, 0LL, 84156416LL, 0LL);//大小为472
v6 = kmem_cache_alloc(cachep, 0x400CC0LL);//从这个kmem_cache中分配一个slot
kmem_cache_free(cachep, filp->private_data);//释放右侧chunk，归入cahchep中。
kmem_cache_destroy(cachep); //摧毁kmem_cache
```

# 2.  Oops泄露kernel地址<br>
触发Oops后的情景如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251023001013.png)
`R10的值ffffffff82a58c20为内核地址段，R12的值0xffff8880043a7000为物理映射区域，但是实际上，它会指向kernel的heap基址！`<br>







<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>