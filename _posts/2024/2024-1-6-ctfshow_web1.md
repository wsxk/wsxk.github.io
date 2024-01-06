---
layout: post
tags: [ctf_wp]
title: "viwe-source"
date: 2024-1-6
author: wsxk
comments: true
---

- [前言](#前言)
- [1. view-source协议](#1-view-source协议)
  - [1.1 ctfshow.web入门.web1](#11-ctfshowweb入门web1)
  - [1.2 题目如何做到的？](#12-题目如何做到的)


## 前言<br>
大学时期主要学习的是二进制安全（re，pwn），到了工作的时候发现，仅仅只是二进制安全还不够，你还需要学习一下web安全，成为全栈选手。这样才能打开你的渗透测试攻击面（我超，泪目）<br>
**这是一个二进制菜鸟学习web安全的记录**<br>
PS: 不知道为什么，攻防世界的题目老是跳容器过期，请重新创建，太tm烦了。<br>

## 1. view-source协议<br>
`view-source协议是一种可以查看网页源代码的协议，可以通过在浏览器地址栏中输入view-source:网址来查看网页源代码，或者按下快捷键Ctrl+U来查看网页源代码。`

<br>
### 1.1 ctfshow.web入门.web1<br>
进入靶场做题时，确实无法看到源码，右键点击或者F12都不行。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240106143223.png)<br>
但是，我们可以通过`view-source:网址`来查看网页源代码。<br>
通过按下`ctrl+u`，即可。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240106143316.png)
这样就拿到flag了。<br>

### 1.2 题目如何做到的？<br>
题目是如何让我们无法右键或者F12呢，仔细观察代码：<br>
有三行javascript代码：<br>
```javascript
<script type="text/javascript">
	window.oncontextmenu = function(){return false};
	window.onselectstart = function(){return false};
	window.onkeydown = function(){if (event.keyCode==123){event.keyCode=0;event.returnValue=false;}};
</script>
```
这段代码就是无法右键/F12的原因。<br>
其中：<br>
**window.oncontextmenu = function(){return false};限制了上下文菜单的显示，即鼠标右键无法出现菜单**<br>
**window.onselectstart = function(){return false};禁止用户选择网页文本，即无法通过鼠标点击和拖动来选择文本**<br>
**window.onkeydown = function(){if (event.keyCode==123){event.keyCode=0;event.returnValue=false;}};禁止用户通过F12来打开开发者工具**<br>