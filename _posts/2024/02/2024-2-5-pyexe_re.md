---
layout: post
tags: [re]
title: "pyinstaller 逆向"
date: 2024-2-5
author: wsxk
comments: true
---

- [前言](#前言)
- [1. 曾经的方案](#1-曾经的方案)
  - [1.1 pyinstxtractor](#11-pyinstxtractor)
  - [1.2 uncompyle6反编译](#12-uncompyle6反编译)
- [2. 现在的方案pydumpck](#2-现在的方案pydumpck)


## 前言<br>
不知道大家看到这张图片是否熟悉<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240221200006.png)
这张图片常出现在**python代码通过pyinstaller打包成exe可执行程序，exe程序的图标**<br>
如果有逆向的同学们应该就知道这个。<br>
**作为逆向的一个常规套路，当然有常规的解题手段，遇到这种开头的exe文件，直接用pyinstxtractor解包，再用python反编译工具把pyc反编译成py文件**<br>

## 1. 曾经的方案<br>
**注意，该方案在使用过程中需要确保你正在运行的python版本，和目标exe文件打包时的python版本是一致的（3.x版本一致），反编译出正确值**<br>
如何获取exe文件打包时的python版本？在经过1.1步骤时也能得知。<br>

### 1.1 pyinstxtractor<br>
[https://github.com/extremecoders-re/pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor)<br>
下载这个工具后，根据提示运行<br>
```
python pyinstxtractor.py <filename>
```
即可得到解压的目录。<br>
**需要注意的是，pyinstxtractor在解压过程中会列出可能的入口函数的pyc文件，之后反编译优先反编译它们即可**<br>

### 1.2 uncompyle6反编译<br>
作为若干年前的神奇，`uncompyle6`给了我很多帮助，然而，`uncompyle6`只支持到`python3.9`版本，在`python3.10`版本后，需要使用其他工具<br>
```
pip install uncompyle6
```
安装完成后，使用<br>
```
uncompyle6 target.pyc > target.py
```
即可<br>

## 2. 现在的方案pydumpck<br>
[https://github.com/serfend/pydumpck](https://github.com/serfend/pydumpck)<br>
**一键式的解决方案，让反编译变得简单轻松**<br>
```
pip install pydumpck
```
后，使用如下命令即可轻松编译<br>
```
pydumpck xxx.exe
```
得到的结果中含有的`pyc`文件也会自动反编译响应的`python`文件，方便查看~<br>