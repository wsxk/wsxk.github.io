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
- [3. 修改smali汇编](#3-修改smali汇编)
  - [3.1 np管理器修改smali代码](#31-np管理器修改smali代码)
- [4. 去广告](#4-去广告)
  - [4.1 广告类型](#41-广告类型)



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


# 3. 修改smali汇编<br>
android应用通常运行在dalvik虚拟机/art虚拟机当中，文件格式为dex（可以类比linux的elf），汇编代码为smali（类比x86）。<br>
修改smali汇编的关键点在于快速定位代码位置，有**搜索关键字**和**抓取按钮id**两种方法。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260325231732.png)

## 3.1 np管理器修改smali代码<br>
定位到位置后，np管理器可以帮助我们方便的修改smali代码<br>
修改smali代码时有几个关键点需要注意：<br>
**.register xx标志了v（局部寄存器）和p（参数寄存器）的总数，且p总是在v的后面**<br>
**在smali里的所有操作都必须经过寄存器来进行:本地寄存器用v开头数字结尾的符号来表示，如v0、 v1、v2。 参数寄存器则使用p开头数字结尾的符号来表示，如p0、p1、p2。特别注意的是，p0不一定是函数中的第一个参数，在非static函数中，p0代指“this"，p1表示函数的第一个 参数，p2代表函数中的第二个参数。而在static函数中p0才对应第一个参数（因为Java的static方法中没有this方法）**<br>
修改完后保存，np管理器就会自动帮忙重新编译并打包签名。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260325234151.png)

# 4. 去广告<br>
## 4.1 广告类型<br>
广告类型有**启动广告（点开qq音乐有时候会有广告跳转到京东、淘宝等等）、弹窗/更新广告（弹个窗出来）、横幅广告（插在应用中**<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>