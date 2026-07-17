---
layout: post
tags: [kernel_pwn]
title: "kernel heap 3: 内核堆利用技巧综述 2"
author: wsxk
date: 2026-7-25
comments: true
---


- [5.3  kaslr + randomized freelist](#53--kaslr--randomized-freelist)


PS： 章节承接[https://wsxk.github.io/kernel_heap2/](https://wsxk.github.io/kernel_heap2/)<br>

## 5.3  kaslr + randomized freelist<br>
攻击条件: 可以任意读写某个 kernel slab的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slab的 `uaf` `double free`<br>
泄露地址:<br>










<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>