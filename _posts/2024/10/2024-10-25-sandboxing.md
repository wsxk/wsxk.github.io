---
layout: post
tags: [pwn]
title: "sandboxing"
author: wsxk
date: 2024-10-25
comments: true
---

- [1. sandboxing由来](#1-sandboxing由来)

## 1. sandboxing由来<br>
`sandboxing`，俗称`沙箱`，是一个在现在看来非常普遍前有效的安全防御措施（比如chrome浏览器里有沙箱，docker也算一种沙箱，etc）<br>
沙箱的诞生也是尤其历史原因的。<br>
```
1. 起始时，大约1950年，计算机中的一切都运行在bare metal（直接在逻辑硬件上执行指令而无需操作系统的计算机）上
那时候，每个进程都是omnipotent（万能的），想干啥就干啥。这就有安全问题：一个进程出问题，整个机器都会出问题

2. 大约1960年，硬件措施被研发出来，用于隔离os代码和用户空间代码
这也有弱点，运行在操作系统的进程仍然可以互相影响

3. 大约1980年，虚拟内存地址空间技术出现，每个进程的内存空间被隔离开来

4. 大约1990年，in-process seperation也流行开来，最主要的做法是隔离解释器和被解释代码
例如，java解释器和java代码，python解释器和python代码

5. 大约2000年，浏览器的攻击手法丰富了起来，当时主要围绕三个浏览器特性进行攻击：
    Adobe Flash
    ActiveX
    Java Applets
通过这些特性，攻击者需要让受害者访问恶意网址，触发漏洞，就能控制受害者系统

6. 大约2010年，浏览器也出了很多的缓解措施。
比如直接关闭这些特性
像Adobe Flash、ActiveX、Java Applets都不允许被使用了
但是这出现了问题：攻击者把目光投向了浏览器的其他特性：
JS engine(JS解析器)漏洞、Media Codec(音频编解码器)漏洞，Imaging library(图像库)漏洞

有些有识之士提出的解决方案是不受信任的代码/数据应该存活在进程的zero-permissions状态下
大致方法是发起父子进程，父进程为高权限进程，子进程为不受信任进程（存放不受信任代码/数据），子进程每次需要运行权限操作，都要经过父进程同意,沙箱应运而生
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241020192813.png)<br>
**沙箱是一个非常强力的缓解措施**，它直接导致了：<br>
```
1. 需要一系列漏洞以利用沙盒进程
2. 需要另一系列漏洞来打破沙盒
```

