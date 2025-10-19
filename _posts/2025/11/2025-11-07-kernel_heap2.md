---
layout: post
tags: [kernel_pwn]
title: "kernel heap 2: practicing"
author: wsxk
date: 2025-11-07
comments: true
---

- [1. kernel heap使用范式](#1-kernel-heap使用范式)



# 1. kernel heap使用范式<br>
```c
cachep = (kmem_cache *)kmem_cache_create("kheap_obj", 472LL, 0LL, 84156416LL, 0LL);//大小为472
v6 = kmem_cache_alloc(cachep, 0x400CC0LL);//从这个kmem_cache中分配一个slot
```








<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>