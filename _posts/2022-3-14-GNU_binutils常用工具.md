---
layout: post
title: "GNU binutils常用工具"
date:   2022-3-14
tags: [linux]
comments: true
author: wsxk
---

## 综述
GNU bintuils是一个二进制工具集，一般情况下linux发行版中都会集成这些工具，换句话说就是，你只要安装了linux系统（物理机 or 虚拟机），里面都会自带这些工具，这些工具你可以在terminal里直接使用。

接下来会介绍一下常用的命令和比较常见的用法

## 常用命令

readelf:读取elf文件信息的

    readelf -h program

ldd:显示程序链接的动态库

    ldd program

size:显示目标文件和可执行文件的节大小

    size program

strings:显示出一个文件中的可打印字符串

    strings -d program

objdump:可以转储目标文件的汇编代码

    objdump -d program

strip：剥离符号，不易于你调试（

    strip program

nm:显示符号（无法显示strip后的程序的符号，因为已经被剥离了）

    nm program

