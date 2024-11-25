---
layout: post
tags: [pwn]
title: "sandboxing —— escape"
author: wsxk
date: 2024-11-25
comments: true
---

- [1. chroot escape](#1-chroot-escape)
  - [1.1 相对路径逃逸](#11-相对路径逃逸)
- [2. namespaces escape](#2-namespaces-escape)
- [3. seccomp escape](#3-seccomp-escape)


## 1. chroot escape<br>
chroot详情可看[https://wsxk.github.io/sandboxing/](https://wsxk.github.io/sandboxing/)<br>

### 1.1 相对路径逃逸<br>
前文提到，chroot是改变了"/"在程序的根目录，这意味着**绝对路径的访问会被限制**，但是相对路径(这取决于你是在哪个目录下运行程序的)是可以绕过这个机制的<br>
示例如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241125213653.png)
上述场景中:<br>
```
1. 真flag 位于 /flag
2. 假flag 位于 /tmp/jaio-eyqnjg/flag
3. 程序的当前工作目录是 /home/wsxk/Desktop/CTF/sandboxing
```
因此绕过时使用的是`../../../../../flag`<br>

## 2. namespaces escape<br>


## 3. seccomp escape<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>