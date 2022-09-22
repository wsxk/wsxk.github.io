---
layout: post
title: "2022HWS冬令营 BadPDF wp"
date:   2022-1-29
tags: [ctf_wp]
comments: true
author: wsxk
---

其实是一道misc题目（但是好玩就玩了下

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/1.png)

打开看了一下发现是个ink文件（快捷方式文件）

打开文件后，发现了有趣的东西（记事本打开）
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/2.png)

这段快捷方式其实运行的是某段代码。提取后如下（记事本就可以提取，再根据bat文件的语法分行即可）

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/3.png)


接下来照着这个代码自己运行一次
然后依次点开每个文件看看是什么东西。
主要注意的是expand.exe指令 这是windows中的解压缩包指令。
所以
oGhPGUDC03tURV.tmp文件是个压缩包。

用7z解压得到
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/4.png)

其中 20200308开头的文件就是我们浏览到的pdf文件。
9s开头的是js脚本，其实就是我们执行的那个命令

cS开头的就是重头戏：应该跟flag有关
同样用记事本打开
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/5.png)

可以看到，主要执行代码是VBS程序，重点是对那串字符串每两个组成一个字符，然后异或1

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-1-29-2022HWSwinter_BadPDF/6.png)

这样，我们就得到了flag