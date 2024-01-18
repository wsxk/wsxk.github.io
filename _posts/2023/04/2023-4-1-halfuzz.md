---
layout: post
tags: [fuzz]
title: "HALucinator: Firmware Re-hosting ARTIFACT Through Abstraction Layer Emulation"
date: 2023-4-1
author: wsxk
comments: true
---

- [install python](#install-python)
  - [安装python](#安装python)
  - [安装python3虚拟环境](#安装python3虚拟环境)
- [安装hal\_fuzz](#安装hal_fuzz)
- [hal-fuzz](#hal-fuzz)
  - [1. 论文思路](#1-论文思路)
  - [2. 实现方法](#2-实现方法)
  - [3. 前置条件](#3-前置条件)
  - [4. 实现细节](#4-实现细节)
- [references](#references)


## install python<br>
**环境 ubuntu18.04**<br>
### 安装python<br>

    sudo apt install python
    sudo apt install python-pip
    sudo apt install python3
    sudo apt install python3-pip

### 安装python3虚拟环境<br>

    sudo pip3 install virtualenv
    sudo pip3 install virtualenvwrapper

在普通用户下，进入

    gedit ~/.bashrc

在末尾加入内容

    export WORKON_HOME=$HOME/.virtualenvs
    export VIRTUALENVWRAPPER_PYTHON='/usr/bin/python3.6'
    source /usr/local/bin/virtualenvwrapper.sh  #实际安装的virtualenvwrapper.sh的路径，可以用 which virtualenvwrapper.sh查看

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230401132130.png)

然后使用如下命令

    source ~/.bashrc


## 安装hal_fuzz<br>
官网在这里[https://github.com/ucsb-seclab/hal-fuzz](https://github.com/ucsb-seclab/hal-fuzz)<br>
首先从github上clone hal-fuzz下来，然后<br>

    mkvirtualenv -p /usr/bin/python3 halfuzz
    sudo ./setup.sh

## hal-fuzz<br>
论文名称为`HALucinator: Firmware Re-hosting ARTIFACT
Through Abstraction Layer Emulation`<br>
### 1. 论文思路<br>
发现各个硬件制造产商提供了一个叫做`Hardware Abstraction Layers (HALs)`的东西。<br>
**HALs是提供给编程师的软件库，其提供了高级别的硬件操作，同时隐藏固件执行时 各个芯片 或 系统的执行细节。这使得程序员的代码移植变得容易起来**<br>
论文作者发现，基于HALs的固件在设计上，更加与硬件脱钩，因此可以方便的实现模拟。<br>
作者想到，结合**HALs和可重复使用的替代功能，即HLE（High Level Emulation）,可是实行高效率的firmware fuzz**<br>
### 2. 实现方法<br>
首先识别 在firmware中的 那些和硬件交互的 HAL 函数<br>
其次 提供一个简单的，由分析师创建的，高级别的替代函数（可以提供和HAL函数在概念上一致的功能）<br>
**值得注意的是，为了找到硬件交互的HAL函数，源码是需要的（至少也要有带符号的firmware）**<br>
找到了这些交互函数后，用自己实现的函数进行替代。<br>

### 3. 前置条件<br>
> 1. 需要事先知道firmware的架构和内存布局（比如Flash和Ram的位置）（可以在手册上找到）
> 2. 已经获得了要模拟的固件所需的底层库，例如 OS library，middleware或者networking stacks，还有固件制造商的工具编译链来编译它们（官网有）
> 3. 一个用于执行模拟指令的程序，例如qemu

总结一下，需要拿到firmware，知道生产商，获得它们的SDK<br>

### 4. 实现细节<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230417125459.png)


***(1) locate the HAL library functions in the firmware (e.g., via library matching)*** <br>
这一步很重要，因为firmware都是完整的，所以firmware自然包含 HAL library中的function，我们要做的是，firmware和 HAL library中的函数进行比对，找到匹配的函数<br>
但是这一步有很多困难： 一般HAL library的函数在编译时会被优化，可能和HAL library中生成的代码不一致；有一些小型的HAL library函数会被预处理；而且一些 HAL library函数还会调用应用程序函数，可能还是可以重写的，十分麻烦<br>
论文作者使用的方法是：<br>
对HAL library进行编译，后生成的库中的每一个函数收录CFG和IR表示，造一个数据库<br>
第一步:  Statistical comparison,通过比对 基本块的数量，CFG边和函数调用，去除不必要的<br>
第二步： Basic Block Comparison，用第一步的输出作为输入，比较两个函数的IR表达是否一致，然而，这个比对并不包括一些指针和相对偏移，以及函数调用等等<br>
第三步： Contextual Matching，用第二步的输出作为输入，因为会有一些错误匹配，因此借助目标函数在firmware中的调用上下文来消除歧义。通过在程序中使用匹配来查找位置来推断其函数可能是什么。使用caller context and callee context<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230417193340.png)
说白了，就是用firmware中这个函数中调用的所有函数名称集合，和匹配到HAL library函数中调用的所有函数名称集合进行比对。再用 firmware中这个函数被调用的所有函数的名称集合，和匹配到的HAL library函数被调用的所有函数名称集合进行比对<br>
第四步： The Final Match，A valid match is identified if a unique
name is assigned to a given function in the target binary<br>

***(2) provide high-level replacements for HAL functions*** <br>
***(3) enable external interaction with the emulated firmware*** <br>


## references<br>
[https://www.dgrt.cn/news/show-128188.html?action=onClick](https://www.dgrt.cn/news/show-128188.html?action=onClick)<br>