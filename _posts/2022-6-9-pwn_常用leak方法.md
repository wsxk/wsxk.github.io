---
layout: post
tags: [pwn]
author : wsxk
date : 2022-6-9
comments : true
title : "pwn中常见leak方式和攻击手段"
---

最近发现，做pwn题的时候，有时侯不是不理解思路，而是有些操作没学会，导致你在思考的时候没有思考全面，容易进入卡死的状态。（你所学到的限制了你的想象力）所以在这里记录一些常见的小tip，方便你泄露一些地址

- [leak方法](#leak方法)
  - [libc中泄露stack](#libc中泄露stack)
  - [mmap申请的chunk泄露libc](#mmap申请的chunk泄露libc)
  - [stripped的libc中找main_arena位置](#stripped的libc中找main_arena位置)
- [攻击思路](#攻击思路)
  - [heap攻击<br>](#heap攻击)
  - [获得栈地址进而ROP](#获得栈地址进而rop)
  - [获得shell后尝试拿到flag<br>](#获得shell后尝试拿到flag)
    - [1.python命令import flag<br>](#1python命令import-flag)
    - [2. cpp flag -o > /dev/null<br>](#2-cpp-flag--o--devnull)
    - [3. ccl flag -o > /dev/null<br>](#3-ccl-flag--o--devnull)
    - [4. as](#4-as)
    - [5. ld](#5-ld)

# leak方法

## libc中泄露stack

libc中有一个叫做 _environ 的全局变量，里面存放着当前进程的环境变量。

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-5-apparmor%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6/20220609214521.png)

从这个图中我们可以看出， _environ存在在libc中，而其中所指向的值，是stack中的。

在IDA分析libc.so中，export符号里会有一个叫environ的符号，可用看它得到相对于libc的偏移

## mmap申请的chunk泄露libc

mmap的chunk地址其实也在libc的那个段中，如果我们可用mmap一个chunk并泄露其堆地址，其实相当于泄露了libc地址。

## stripped的libc中找main_arena位置

在ida拖入libc后，搜索函数malloc_trim

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-5-apparmor%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6/20220612210734.png)

这个高亮即为main_arena的地址

其中main_arena有几个偏移比较重要，需要记住

在main_arena+112处，是bins数组的起始位置。因为一开始bins里什么都没有，所以指向main_arena+112（空bins都是这个，所以有很多），如果能泄露这个位置的值，就可以泄露libc

在mian_arena+96处，存放的是top_chunk的地址，据此可以推出heap的地址。

# 攻击思路
## heap攻击<br>
首先可以好好看看how2heap里的利用手法。

## 获得栈地址进而ROP

这个在最新的堆里面是很常见的（2.34）

## 获得shell后尝试拿到flag<br>
一般情况下cat命令无法使用，重点在于利用其他命令执行sys_open来读取flag<br>
### 1.python命令import flag<br>
### 2. cpp flag -o > /dev/null<br>
### 3. ccl flag -o > /dev/null<br>
### 4. as
### 5. ld
