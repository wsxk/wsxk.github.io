---
layout: post
tags: [software_build]
title: "ubuntu 安装AFL"
date: 2023-3-1
author: wsxk
comments: true
---


AFL的安装还是比较简单的。<br>
首先去官网下载安装包:<br>
[https://lcamtuf.coredump.cx/afl/](https://lcamtuf.coredump.cx/afl/)<br>

然后用如下命令进行解压<br>
```shell
tar -xvf afl-latest.tgz
```
进入解压目录<br>
```shell
cd afl-2.52b
make
sudo make install
```

即可