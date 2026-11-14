---
layout: post
tags: [linux]
title: "linux Ⅲ：CIA、Bash魔术&潜在的攻击方法和防御手段"
author: wsxk
date: 2026-11-28
comments: true
---


- [1. 安全三要素(CIA)](#1-安全三要素cia)
- [2. linux bash魔术](#2-linux-bash魔术)
  - [2.1 program和builtin的帮助指南](#21-program和builtin的帮助指南)
  - [2.2 好用的命令：diff](#22-好用的命令diff)
  - [2.3 重定向](#23-重定向)
  - [2.4 管道](#24-管道)


# 1. 安全三要素(CIA)<br>
安全三要素指:<br> 
**机密性（confidentiality）**：你的机密信息有没有被窃取<br>
**完整性（integrity）**：你的数据有没有被篡改<br>
**可用性（availability）**：服务是否还能响应<br>

# 2. linux bash魔术<br>
## 2.1 program和builtin的帮助指南<br>
先讲一下program和builtin的区别，program是linux系统里的程序（有文件承载，如ls），builtin是linux中shell内置的命令（如cd）。<br>
主要有三种方式:<br>
```
man program
program --help
help builtin
```
可以通过```type xxx```查看命令所属类别。<br>

## 2.2 好用的命令：diff<br>
比对不同文件的好命令，常规用法:`diff a b`<br>
```shell
diff a.sh x.sh 
1c1,3   # 左边的1表示 a.sh的第1行发生了更改，更改为x.sh的第1~3行
< /challenge/solve
---
> 
> /home/hacker/linux/x.sh &
> /home/hacker/linux/x.sh &
```


## 2.3 重定向<br>
每个默认的linux程序，要输出东西或者输入东西，都是通过文件描述符（内核中IO操作结构体指针的索引），每个程序默认都会有三个标准的文件描述符，代表标准输入（0）、标准输出（1）、标准错误（2）。 我们可以通过shell的重定向功能，修改标准输入所代表的IO操作结构体，达成更换读取/写入位置的目的。<br>
linux用法:<br>
```
command <file1  # < 以只读方式打开file1，并把command的标准输入（0）重定向到这个文件
command >file1  # > 以只写方式打开file1，并把command的标准输出重定向到这个文件
command 2>&1    # >& 把fd1重定向到fd2
command 2<&1    # <& 把fd1重定向到fd2  规则: 左边的 FD 被修改，右边的 FD 是复制来源
```

## 2.4 管道<br>



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>