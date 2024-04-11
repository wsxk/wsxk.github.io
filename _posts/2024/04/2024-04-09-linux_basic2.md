---
layout: post
tags : [linux]
date: 2024-4-9
title: "linux: 程序加载与执行"
author: wsxk
comments: true
---

- [前言](#前言)
- [11. program加载与执行过程](#11-program加载与执行过程)
  - [11.1 进程创建](#111-进程创建)
  - [11.2 进程加载](#112-进程加载)
  - [11.3 进程初始化](#113-进程初始化)


## 前言<br>
想要了解linux的基础概念，可以先看看[https://wsxk.github.io/linux_basic/](https://wsxk.github.io/linux_basic/)

## 11. program加载与执行过程<br>
当你在一个`shell`中执行一个程序时，你不会好奇：**这个程序是如何被加载然后执行的吗？**<br>
**以执行/bin/cat为例，程序会执行如下7个步骤**<br>
```
1. A process is created. 进程创建
2. Cat is loaded. Cat程序被加载
3. Cat is initialized. Cat程序被初始化
4. Cat is launched.  Cat程序被运行
5. Cat reads its arguments and environment. Cat程序读取参数和环境变量
6. Cat does its thing. Cat程序开始做正式工作
7. Cat terminates.  Cat程序结束运行
```
### 11.1 进程创建<br>
在`linux`系统中，进程都是通过分裂来进行传播的，具体而言，当我们在`terminal(bash)中执行cat xxx`时，`父进程(bash)`会通过系统调用**fork()或者clone()**创建跟父进程近乎一样的子进程，随后，子进程会通过系统调用**execve()**来把自身替换成其他进程，在这个例子中就是`cat`<br>

### 11.2 进程加载<br>
在执行**execve()**这个动作时，`kernel`会检测文件是否可执行，即`executable权限`，如果不可执行，那么**execve()系统调用**会失败<br>
在确定文件是可执行的后，`kernel`还会为了确定加载什么内容而做如下图所示的检测<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240409231213.png)

> - 1 首先判断文件是否以#!开头，如果是，kernel会提取该行接下来的内容，并将其当作解释器用来执行，原始的命令作为解释器的参数（直接跟在解释器后面）

例子： <br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240409232105.png)
在这个例子,文件以`#!开头，kernel提取/bin/echo作为解释器，即运行这个程序`<br>
此时命令相当于：<br>
        
    /bin/echo ./some-script

因此`some-script`文件中的`echo hi`就不会打印出来<br>
这个过程也可以是递归的，再看一个例子：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240409232450.png)
在这个例子中,`some-script2`中的解释器是`some-script`，在执行命令`./some-script2`时，相当于:<br>

    ./some-script ./some-script2

而在运行`./some-script ./some-script2`相当于<br>

    /bin/echo ./some-script ./some-script2
    
十分神奇！<br>

> - 2 如果文件的格式满足 /proc/sys/fs/binfmt_misc中的内容，kernel会执行特定格式对应的解释器，原始的命令作为解释器的参数

举个例子:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240410220458.png)
在这里，**如果文件的开头是550d0d0a，那么就用/usr/bin/python3.8来执行这个文件**<br>

> - 3 如果文件是动态链接的elf文件，kernel会选定elf文件中loader定义的值来作为解释器，加载loader和原始文件，并让loader来进行控制

**loader是很重要的，如果elf文件是动态链接的，loader会负责so的地址分配，符号解析，重定位，合并段....等等用途，保证elf能够顺利执行**<br>
`loader可以通过 readelf -a /bin/cat | grep interpreter`来查询<br>
下面这张图也能体现`动态链接elf的加载过程`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240410222710.png)

例子又来了:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240411220850.png)
用`gcc -shared -o preload.so preload.c`将其编译出`so文件`<br>
我们可以用`strace -E LD_PRELOAD=./preload.so ./cat cat.c 2>&1 | head -n 100`来查看先后调用关系<br> 
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-04-11%20221232.png)

如果用的是`strace -E LD_LIBRARY_PATH=/some/lib ./cat cat.c`来追踪的话：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240411221615.png)
其他的路径可以自行实验~<br>


> - 4 如果文件是静态链接的elf，kernel会直接加载它

很直接，没有啥好说的<br>

> - 5 其他遗留的文件格式会被检查

这个也很直接<br>

### 11.3 进程初始化<br>
在所有文件都加载完毕后，自动执行的函数通常是：``<br>