---
layout: post
tags: [pwn]
title: "glibc 2.35 thread-heap: arenas 利用手法"
author: wsxk
date: 2025-5-25
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

# 1. arbitrary read<br>

# 2. arbitrary write<br>




<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>