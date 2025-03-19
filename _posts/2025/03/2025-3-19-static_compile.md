---
layout: post
tags: [re]
title: "静态程序符号还原方法"
author: wsxk
comments: true
date: 2025-3-19
---


- [0. 写在前面](#0-写在前面)
- [1. sig-database](#1-sig-database)
- [references](#references)

# 0. 写在前面<br>
最近在做re、pwn题时，经常遇到一些静态编译过后，又去除符号的程序。<br>
对于动态编译的去符号程序，emm，去除的符号通常都是作者自定义的函数名称，你想通过工具还原其符号通常情况下是不太可能的。但是这种情况下，做符号还原的常规办法还是理解代码逻辑后为其命名，通常不会带来太大的工作量（毕竟你做题也需要理解代码逻辑<br>
但是对于**静态编译的去符号程序，问题就来了：库函数和程序的主逻辑混杂在一起，让逆向工作者无法判断该函数是程序函数，还是库函数，无法聚焦程序核心逻辑的分析。**<br>
为了应对这个情况，首要思路还是想办法还原库函数，将库函数和程序代码函数分离，帮助我们更快解题<br>

# 1. sig-database<br>
sig-database顾名思义，是一个面向ida pro的符号存储库，其存储了绝大部分程序会使用的so的sig，我们可以直接将sig-database中的sig中下载下来，项目地址:[https://github.com/push0ebp/sig-database](https://github.com/push0ebp/sig-database)<br>
下载下来后，我们可以把里面存放的sig文件，转存到`ida目录下的sig/pc目录下`。<br>
存放完成后，在ida使用界面中点击`File->Load file-> FLIRT signature file`将so文件导入，可以一次性多导入几个，总会有几个成功的（😀<br>


发现新的好办法再继续更新<br>

# references<br>
[恢复静态编译去符号](https://noone-hub.github.io/posts/fea8bac7/)<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>