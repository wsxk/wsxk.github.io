---
layout: post
tags: [linux]
title: "linux Ⅲ：CIA、Bash魔术&潜在的攻击方法和防御手段"
author: wsxk
date: 2026-11-15
comments: true
---


- [1. 安全三要素(CIA)](#1-安全三要素cia)
- [2. linux bash魔术](#2-linux-bash魔术)
  - [2.1 program和builtin的帮助指南](#21-program和builtin的帮助指南)
  - [2.2 好用的命令：diff](#22-好用的命令diff)
  - [2.3 重定向](#23-重定向)
  - [2.4 管道](#24-管道)
  - [2.4.1 process substitution](#241-process-substitution)
  - [2.5 变量](#25-变量)
  - [2.6 数据操作](#26-数据操作)
  - [2.7 进程管理](#27-进程管理)
- [3. 用户权限管理](#3-用户权限管理)
- [3.1 su（switch user）](#31-suswitch-user)
- [3.2 sudo(superuser do)](#32-sudosuperuser-do)
- [4. 文件/目录权限管理](#4-文件目录权限管理)
  - [4.1 权限更改操作](#41-权限更改操作)
  - [4.2 suid](#42-suid)
  - [4.3 t(sticky bit)](#43-tsticky-bit)
- [5. tmux用法](#5-tmux用法)
- [6. 潜在的攻击](#6-潜在的攻击)
  - [6.1 针对PATH的攻击](#61-针对path的攻击)
  - [6.2 bashrc后门](#62-bashrc后门)
  - [6.3 进程信息泄露、bashrc信息泄露](#63-进程信息泄露bashrc信息泄露)
  - [6.4 fork bomb](#64-fork-bomb)
  - [6.5 磁盘空间耗尽](#65-磁盘空间耗尽)
  - [6.6 rm -rf / ](#66-rm--rf--)


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
command 2>&1    # >& 把fd2重定向到fd1
command 2<&1    # <& 把fd2重定向到fd1  规则: 左边的 FD 被修改，右边的 FD 是复制来源
```

## 2.4 管道<br>
顾名思义，管道其实是用来连接两个进程，用于通信的一种结构体。管道有多种类型，在linux shell当中，通常使用`|`来代替，**用来将一个命令的标准输出（1）连接到另一个命令的标准输入（0）**<br>
```
command1 | command2
```

## 2.4.1 process substitution<br>
在shell当中，通常以如下形式出现:<br>
```
diff <(sort a.txt) <(sort b.txt)
```
`<()` `>()`是进程替换的特殊表示，以`cat <(printf 'abc\n')`为例，实际上执行了:<br>
```
1. 创建一个匿名管道
2. 将printf的内容输出到匿名管道中
3. 将 <(printf 'abc\n') 替换为管道读端的路径（例如/dev/fd/63）
```

当然，我们想创建实际的命名管道，可以用`mkfifp path`。<br>
想要便于调试管道之间出了什么问题，可以中间连一个:<br>
```
command1 | tee file | command2
```
`tee`会将其标准输入复制到标准输出以及指定文件。所以标准输出还是会进入command2，但是可以在文件中查看command1的输出。<br>

## 2.5 变量<br>
在linux shell当中，变量的用法如下:<br>
变量 variable：` echo $var, var=abc   export var=$(command) `<br>
```
var=abc # 设置变量var=abc，不能有空格
echo $var # 使用变量时要添加$符号
export var=$(command)  # 把变量从local引入到env当中，这样开启新进程也会继承这个变量。
```

## 2.6 数据操作<br>
数据操作 tr、head、cut、sort<br>

## 2.7 进程管理<br>
在linux shell当中，可以查看当前系统正在运行的进程：<br>
```
ps -ef , ps -aux  # 查看进程信息
kill pid # 杀死进程
ctrl+c # 杀死前台进程
ctrl+z # 将前台正在运行的进程中断
fg # 将中断的进程重新放入前台执行
bg # 将中断的进程放入后台执行
command & # 以后台运行的方式启动程序
echo $?  # 输出上个进程执行的返回值。

```

# 3. 用户权限管理<br>
linux系统中是有管理员(root)和普通用户之分的，管理员可以做任何事。<br>
# 3.1 su（switch user）<br>
普通用户在linux系统里是很受限的，它也需要以root用户执行命令。<br>
`su`默认执行时，会尝试将当前用户切换为root，但是要输入密码。<br>
当然，你也可以用`su user`切换为其他用户，只不过都要输入密码。<br>

实际切换用户时，su会查看`/etc/passwd`（全局可读）和`etc/shadow`（root可读）中读取密码的哈希，用你输入的密码生成哈希与其进行比对。<br>

# 3.2 sudo(superuser do)<br>
用su相当于让普通用户知道root的密码，这样权限就直接分散出去了，为了缓解这个情况，另一个命令sudo出现了。<br>
`sudo`允许让普通用户能够以root的权限执行某些命令（不需要密码）<br>
相关的配置在`/etc/sudoers`文件当中。<br>

# 4. 文件/目录权限管理<br>
文件、目录都有`rwx`三种权限<br>
其含义如下:<br>
```
文件: r（可读） w（可写） x（可执行）
目录：r（可查看，ls） w（可创建、删除文件）、x（可进入目录，cd）
```
针对`owner（创建者）、group（创建者所属群组）、other（其他人）`都有上述三种权限。<br>
## 4.1 权限更改操作<br>
```
chown user file # 更改文件的owner
chgrp group file # 更改文件的group
chmod who+/-/=what, who+/-/=what file #改变文件的权限
```

## 4.2 suid<br>
特殊的文件标志位，被设定到文件中时，如果你有该文件的执行权限，执行时将按照该文件的所有者的权限执行该命令，（su 和sudo也是基于此原理）<br>

## 4.3 t(sticky bit)<br>
特殊的目录标志位，设置到目录上时，目录内的文件，只有文件的拥有者可以**重命名/删除**文件。<br>


# 5. tmux用法<br>
```
tmux          #新建tmux的一个session和一个window中 
tmux ls       #查看哪些session和windows
tmux a        # 重新进入某个window
# 以下均需要进入window后执行
ctrl+b d      # 离开windows
ctrl+b c      # 创建一个新window
ctrl+b n      # 下一个window
ctrl+b p      # 前一个window
ctrl+b w      # 选择window
ctrl+b "      # window上下分屏
ctrl+b %      # window左右分屏
ctrl+b o      # 切换一个window内的pane
```

# 6. 潜在的攻击<br>
## 6.1 针对PATH的攻击<br>
调用命令有3种方式：<br>
```
1. 以绝对路径执行
2. 以相对路径执行
3. 直接命令执行（需要用到PATH的环境变量）
```
如果攻击者能够修改以root权限执行的程序的环境变量的话，相当于任意执行命令

## 6.2 bashrc后门<br>
`~/.bashrc`文件是每个用户，启动bash的最先执行的脚本，如果能够操控别的用户的`~/.bashrc`文件，也能在其他用户执行bash时劫持。<br>

**这通常是一个系统内，为了多个用户方便共享，开放某用户A的私人目录权限为其他人可写（其他用户可以进入用户删除.bashrc文件并重新创建一个.bashrc文件**完成。<br>

另一个方法是在临时的共享目录内具有可写的权限，但是root用户A会修改共享目录内的某个文件x，可能能够**通过ln创建软链接的方式，把文件x执行root用户的.bashrc文件，让root用户执行代码时，修改bashrc自身达成**<br>


通过给共享目录添加`t`粘滞位解决上面遇到的问题。<br>

## 6.3 进程信息泄露、bashrc信息泄露<br>
普通用户通过ps命令查看到root用户进程的执行参数，可能暴露用户名密码。<br>
因为.bashrc默认是可读的，所以可以读取其他用户的.bashrc文件看看里面有没有机密信息。<br>


## 6.4 fork bomb<br>
进程炸弹，linux系统里的破坏性行为，用于耗尽资源。<br>
```bash
/home/hacker/linux/x.sh &
/home/hacker/linux/x.sh &
```

## 6.5 磁盘空间耗尽<br>
用yes命令在创建某个文件并持续写入内容，不久就会占满磁盘空间。<br>


## 6.6 rm -rf / <br>
删除全部文件。




<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>