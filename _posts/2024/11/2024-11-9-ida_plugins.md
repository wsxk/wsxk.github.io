---
layout: post
tags: [software_build]
title: "ida 9.0 rc1 & plugins"
author: wsxk
date: 2024-11-9
comments: true
---


- [0. 前言](#0-前言)
- [1. ida9.0在哪下](#1-ida90在哪下)
- [2. ida plugins](#2-ida-plugins)
  - [2.1 keypatch](#21-keypatch)
  - [2.2 VulFi](#22-vulfi)
  - [2.3 ](#23-)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## 0. 前言<br>
ida9.0 rc1版本已经释放出来，这对安全研究人员是一个非常大的利好！<br>
但是ida9.0 rc1的版本做了一些变动，有些插件不能用了，所以这里着重介绍当前在ida 9.0中可行且好用的插件<br>

## 1. ida9.0在哪下<br>
看雪[https://bbs.kanxue.com/thread-283752.htm](https://bbs.kanxue.com/thread-283752.htm)中有内容，
下载完成后，windows需要做如下操作:<br>
先运行`ida-pro_90_x64win.exe`完成ida9.0rc1的安装，随后<br>
在`kg_patch`目录下找到`idapro.hexlic`和`keygen2.py`文件，拷贝并复制到安装的ida目录下（该目录有ida.exe程序）<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241109215555.png)
在当前目录下打卡终端，运行`python keygen2.py`后，即可使用<br>


## 2. ida plugins<br>
### 2.1 keypatch<br>
非常好用的patch工具，打awd时，patch全靠它，下载地址<br>
[https://github.com/keystone-engine/keypatch](https://github.com/keystone-engine/keypatch)

### 2.2 VulFi<br>
VulFi是一个比较好用的二进制漏洞发现工具，实际体验下来还可以，能够发现一些问题！<br>
[https://github.com/Accenture/VulFi](https://github.com/Accenture/VulFi)

### 2.3 <br>