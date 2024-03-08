---
layout: post
tags : [linux]
date: 2024-3-3
title: "linux basic"
author: wsxk
comments: true
---

- [写在前面](#写在前面)
- [1. Command Line](#1-command-line)
- [2. process、program、filesystem、directories](#2-processprogramfilesystemdirectories)
- [3. absolute/relative path](#3-absoluterelative-path)
- [4. environment varibles](#4-environment-varibles)
- [5. symbolic/hard links](#5-symbolichard-links)
- [6. pipes](#6-pipes)
- [7. input/ouput redircetion](#7-inputouput-redircetion)
- [8. 权限管理](#8-权限管理)
- [9. linux常见命令](#9-linux常见命令)
  - [9.1 linux命令使用说明书](#91-linux命令使用说明书)
  - [9.2 常见命令列表](#92-常见命令列表)


## 写在前面<br>
虽然学习网络安全四年有余了，至今让你提起linux的一些关键概念，还是有些不了解linux系统中的某些关键概念，倍感羞耻<br>

## 1. Command Line<br>
俗称`shell`，是一种与用户交互的界面,本质上也是一个进程（运行中的程序）<br>
用户可以在如下界面输入程序，如`cat flag（寻找叫做cat的程序，以flag作为参数运行）`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303202219.png)

## 2. process、program、filesystem、directories<br>
**process（进程）指运行中的program（程序）**<br>
**program指的是存储在文件系统(filesystem)中的文件**<br>
`linux`中的文件系统布局通常如下图所示<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303202451.png)
**刚刚说program存储在文件系统中，更准确的来说，存储在文件系统的directories(目录)中**<br>

## 3. absolute/relative path<br>
**绝对路径absolute path**指的是文件在系统中的存放位置<br>
**相对路径relative path**指的是文件相对于当前工作目录下的存放位置<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303202736.png)
关于目录，`.`表示当前目录，`..`表示上一级目录，`/`开头的表示根目录（即最初始的目录）<br>
我们可以通过`ls -l`查看文件的类型，常见类型如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303203344.png)

## 4. environment varibles<br>
`environment varibles`是一组键值对（key-value）的集合，在每个程序被执行时，会传递给程序<br>
可以通过`env`程序查看当前的环境变量<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303203058.png)
***但凡我们执行诸如 cat flag这样的命令，然而cat又不在当前目录下，我们也没有输入cat的路径，cat的路径都已经保留在环境变量当中**<br>

## 5. symbolic/hard links<br>
`symbolic links`软链接，指的是`创建一个特殊类型的文件，指向另一个文件`,通过读取这个软链接相当于读取目标文件<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303210124.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303210208.png)
当然，也可以链接`目录`<br>
**注意陷阱：**<br>
**软连接创建时，如果路径使用相对路径时，在使用该软连接进行文件操作时，寻找真的文件时，使用的相对路径会是你当前的动作目录**，所以如果移动了软连接的位置，就用不了了<br>
如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303211454.png)


`hard links`硬链接，与软链接有所不同，创建的是一个指向目标文件的**真实的引用**<br>
如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303211958.png)
从图示中也可以看出，文件类型`-`，表示普通文件<br>
实际原理跟`inode`相关<br>


## 6. pipes<br>
`pipes（管道）`是一种**单向的信息流通机制**<br>
管道分为两种:<br>
```
1. Unnamed pipes（匿名管道）：
最常用于把一个命令的数据 传递到 另一个命令
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303212603.png)
```
2. Named pipes（命名管道）：也被称为 FIFOS
可以使用"mkfifo" 命令来创建
被使用在特定场景下的数据流程
```

## 7. input/ouput redircetion<br>
`输入/输出重定向`指的是 命令的输出不一定就在界面上，可以重定向到文件中， 命令的输入也是，不一定是直接在界面输入，也可以重定向到文件中<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303213020.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303213100.png)

## 8. 权限管理<br>
详情可见[https://wsxk.github.io/linux%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86/](https://wsxk.github.io/linux%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86/)<br>

## 9. linux常见命令<br>
### 9.1 linux命令使用说明书<br>
linux中有很多命令可供选择，为了了解命令的使用方式，了解命令说明书的构造就尤为重要<br>
以某个例子为例<br>
```shell
Usage: cpio [OPTION...] [destination-directory]
有 []攘括的 OPTION 表示，该参数是可选参数（可加可不加）
有 ...跟随，表示OPTION可以有一个或多个

除此之外还有其他常见符号
<> 攘括的参数表明该参数不可缺失，比如 command <filename>
{} 攘括的参数表明必须使用其中一个参数， 比如 command {a | b}，表示a、b二选一，
```

### 9.2 常见命令列表<br>
**常用的获取文本的命令**<br>
```
more ： 浏览文件内容
less ： 浏览文件内容
tail ： 查看文件末尾几行
head ： 查看文件头几行
sort ： 文件行排序后输出
rev ： 倒序输出

vim ： 文本编辑器
emacs ： 文本编辑器
nano ： 文本编辑器

od ： Octal Dump
hd ： Hex Dump
xxd ： 主要用于创建十六进制或二进制的转储

base32: 内容以base32加密后输出
base64: 内容以base64加密后输出

split： 将一个文件的内容 分成若干文件

gzip： 压缩/解压缩 文件 后缀： .gz
bzip2： 压缩/解压缩 文件 后缀： .bz2
zip: 压缩/解压缩文件    后缀： .zip root权限zip压缩后的文件，普通用户也有读权限，可以通过unzip解压
tar： 压缩/解压缩文件 tar -xf flag.tar -O 可以把解压的内容输出到命令行中
ar： 创建、修改静态库 ar -crv flag.a flag 后 使用 ar -p flag.a flag 即可打印flag内容
cpio： 压缩/解压缩 cpio -o <~/1.txt用于创建一个压缩文件

```