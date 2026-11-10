---
layout: post
tags: [linux]
title: "linux Ⅲ：CIA和Bash魔术"
author: wsxk
date: 2026-11-28
comments: true
---


- [1. 安全三要素(CIA)](#1-安全三要素cia)
- [2. linux bash魔术](#2-linux-bash魔术)
  - [2.1 program和builtin的帮助指南](#21-program和builtin的帮助指南)
  - [2.2 好用的命令：diff](#22-好用的命令diff)


# 1. 安全三要素(CIA)<br>
安全三要素指:<br> 
**机密性（confidentiality）**：你的机密信息有没有被窃取<br>
**完整性（integrity）**：你的数据有没有被篡改<br>
**可用性（availability）**：服务是否还能响应<br>

# 2. linux bash魔术<br>
## 2.1 program和builtin的帮助指南<br>
先讲一下program和builtin的区别，program是linux系统里的程序（有文件承载，如ls），builtin是linux中shell内置的命令（如cd）。<br>
主要有三种方式:<br>
```
man program
program --help
help builtin
```

## 2.2 好用的命令：diff<br>






<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>