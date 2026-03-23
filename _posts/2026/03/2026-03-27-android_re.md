---
layout: post
tags: [Android]
title: "android 应用tips"
author: wsxk
date: 2026-03-27
comments: true
---

- [1. 双开](#1-双开)
  - [1.1 双开原理](#11-双开原理)
  - [1.2 双开工具](#12-双开工具)
- [2. 汉化](#2-汉化)
  - [2.1 汉化步骤](#21-汉化步骤)
  - [2.2 汉化工具](#22-汉化工具)



简单记录一下android的使用小技巧。<br>
# 1. 双开<br>
## 1.1 双开原理<br>
安卓系统中通常以一个系统中`包名+用户（使用者身份）空间`来标识应用，所有绕过系统的双开限制，基本上围绕这两点来搞事：<br>

| 原理               | 解释                                                                                                                                     |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 修改包名           |让手机系统认为这是2个APP，这样的话就能生成2个数据存储路径，此时的多开就等于你打开了两个互不干扰的APP                                                                                                                                          |
| 修改Framework      | 对于有系统修改权限的厂商，可以修改Framework来实现双开的目的，例如：小米自带多开                                                            |
| 通过虚拟化技术实现 | 虚拟Framework层、虚拟文件系统、模拟Android对组件的管理、虚拟应用进程管理 等一整套虚拟技术，将APK复制一份到虚拟空间中运行，例如：平行空间 |
| 以插件机制运行  |利用反射替换，动态代理，hook了系统的大部分与system—server进程通讯的函数，以此作为“欺上瞒下”的目的，欺骗系统“以为”只有一个apk在运行，瞒过插件让其“认为”自己已经安装。例如：VirtualApp|

## 1.2 双开工具<br>
`mt管理器和np管理器`,mt管理器的大部分功能都需要付费，所以这里用np管理器试验<br>
[http://normalplayer.top/](http://normalplayer.top/)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260322235404.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260322235458.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260322235550.png)
安装即可。<br>

# 2. 汉化<br>
顾名思义，将他国语言的app转为本国语言，通常**汉化包括asrc汉化、xml汉化和dex汉化**<br>
## 2.1 汉化步骤<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260323215230.png)

## 2.2 汉化工具<br>

在`np`管理器中，对提取的apk进行搜索功能，使用高级搜索，在文件中包含内容里寻找相关字符串：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260323210920.png)
找到字符串位置后，修改即可。<br>
针对看不懂的语言，可以用[开发者助手](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/%E5%BC%80%E5%8F%91%E8%80%85%E5%8A%A9%E6%89%8B.apk)来试图并修改。<br>
用开发者助手提取字符串，然后到`np`管理器里搜索对应的文本：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260323214154.png)

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>