---
layout: post
tags: [kernel_pwn]
title: "kernel stack 2: ret2usr"
author: wsxk
date: 2026-9-10
comments: true
---

- [例题: hxp 2020 kernel-rop](#例题-hxp-2020-kernel-rop)
- [4. ret2usr](#4-ret2usr)


# 例题: hxp 2020 kernel-rop<br>
非常好的试验例题:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260821220047.png)
漏洞也很明显，栈溢出。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260822203851.png)
还有一个栈溢出读。<br>

# 4. ret2usr<br>
本质是劫持内核控制流，使其跳转到用户态的函数并执行。<br>
正如[https://wsxk.github.io/kernel_stack1/](https://wsxk.github.io/kernel_stack1/)提到的，要想使用ret2usr技术，需要关闭`smep、smap、kpti`3个机制的保护才行。<br>








<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>