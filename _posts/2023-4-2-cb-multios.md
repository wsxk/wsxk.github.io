---
layout: post
tags: [fuzz]
title: "cb-multios install"
author: wsxk
date: 2023-4-2
comments: true
---

- [introduction](#introduction)
- [install](#install)
  - [1. environment](#1-environment)
  - [2. prerequisites](#2-prerequisites)
  - [3. follow the guide](#3-follow-the-guide)
  - [4. testing](#4-testing)


## introduction<br>
cb-multios是**评估程序分析工具/漏洞探寻工具的基准测试**<br>
其中包含了许多内置了漏洞的，仿真的程序，可供程序分析软件程序分析工具/漏洞探寻工具进行测试，说白了就是为了测试工具的有效性，性能，等等。<br>
当然fuzz测试也可以利用这个项目。<br>

## install<br>
### 1. environment<br>
系统：ubuntu 18.04

### 2. prerequisites<br>

    sudo apt install libc6-dev libc6-dev-i386 gcc-multilib g++-multilib clang

注意不要使用`sudo apt install cmake`来安装`cmake`程序，因为版本太低了，因此你需要自己去官网下载[https://cmake.org/download/](https://cmake.org/download/)<br>
下载后解压即可，随后你需要把cmake放入你的环境当中，使得可以使用:

    ln -s /path/to/your/cmake /usr/bin/cmake

### 3. follow the guide<br>
跟随cb-multios开源项目进行安装即可。<br>

### 4. testing<br>
开源项目同样提供了测试工具，判断程序能否正确运行，因此，just use it。<br>
