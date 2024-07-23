---
layout: post
tags: [pwn]
title: "pwntools+tmux"
date: 2024-7-23
author: wsxk
comments: true
---

- [1. why pwntools+tmux?](#1-why-pwntoolstmux)
- [2. 安装教程](#2-安装教程)
- [3. 使用教程](#3-使用教程)
- [4. tmux快捷键](#4-tmux快捷键)


## 1. why pwntools+tmux?<br>
在用`pwntools`进行`ctf pwn`题目调试时，通常会利用类似`gdb.attach(p)`的代码来调试，通常情况下，这回弹出另一个命令行<br>
**但是你用tmux来进行调试，它会把一个命令行划分成2个pane，可以同时操作，界面美观还方便**<br>

## 2. 安装教程<br>
对于`pwntools`而言，只需执行如下命令即可。<br>
```
pip install pwntools
```
对于`tmux`，在`ubuntu20`虚拟机下，执行如下命令：<br>
```
sudo apt install tmux 
```

## 3. 使用教程<br>
开启命令行后，使用如下命令:<br>
```shell
tmux
```
这会开启一个`tmux`的服务端<br>
然后，python编写如下代码:<br>
```python
from pwn import *
log.level = "debug"
context.terminal=["tmux","splitw","-h"]

io = process(["./cts","8888"])
p = remote("127.0.0.1","8888")
gdb.attach(io)

p.interactive()
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240723215924.png)
可以看到，使用`gdb.attach(io)`后，直接从当前`terminal`分成2个，方便调试。<br>

## 4. tmux快捷键<br>
`tmux`情境下,使用命令前都需要打出`ctrl+b`作为前缀，为了切换键盘输入，你需要使用`ctrl+b o`<br>
[https://matpool.com/supports/doc-tmux-matpool/](https://matpool.com/supports/doc-tmux-matpool/)里有大量的`tmux命令教程`<br>