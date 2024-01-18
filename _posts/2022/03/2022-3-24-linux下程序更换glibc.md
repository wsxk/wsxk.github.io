---
layout: post
title: "linux下程序更换glibc"
date:   2022-3-24
tags: [pwn]
comments: true
author: wsxk
---

linux下程序更换glibc的方法有很多，更换glibc的办法有很多，但是步骤都很相似：下载glibc,ld 然后更换程序的glibc，ld

## 下载glibc

[https://wsxk.github.io/glibc%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91/](https://wsxk.github.io/glibc%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91/)

推荐使用源码安装，因为看得方便~~，debug也方便

## 更换glibc

在这里使用python的pwntools库中的

    io=process("./test",env={"LD_PRELOAD":"/path/to/glibc"})  #正常情况下这个就行了，但是有时候可能会出问题

    io=process(["/path/to/ld.so", "./test"],env={"LD_PRELOAD":"/path/to/glibc"}) #有点小问题，gdb.attach()时，attach的是ld，十分不方便。

也有另一种方式

    patchelf --set-interpreter ld.so binary

然后直接用process("./binary")即可



