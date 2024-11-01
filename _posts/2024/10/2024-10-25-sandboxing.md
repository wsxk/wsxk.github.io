---
layout: post
tags: [pwn]
title: "sandboxing"
author: wsxk
date: 2024-10-25
comments: true
---

- [1. sandboxing由来](#1-sandboxing由来)
- [2. chroot](#2-chroot)
  - [2.1 chroot使用注意事项](#21-chroot使用注意事项)
  - [2.2 chroot作用](#22-chroot作用)
  - [2.3 chroot陷阱](#23-chroot陷阱)
  - [2.4 chroot安全性讨论](#24-chroot安全性讨论)
- [3. seccomp](#3-seccomp)
  - [3.1 seccomp示例](#31-seccomp示例)
  - [3.2 seccomp工作原理](#32-seccomp工作原理)

## 1. sandboxing由来<br>
`sandboxing`，俗称`沙箱`，是一个在现在看来非常普遍前有效的安全防御措施（比如chrome浏览器里有沙箱，docker也算一种沙箱，etc）<br>
沙箱的诞生也是尤其历史原因的。<br>
```
1. 起始时，大约1950年，计算机中的一切都运行在bare metal（直接在逻辑硬件上执行指令而无需操作系统的计算机）上
那时候，每个进程都是omnipotent（万能的），想干啥就干啥。这就有安全问题：一个进程出问题，整个机器都会出问题

2. 大约1960年，硬件措施被研发出来，用于隔离os代码和用户空间代码
这也有弱点，运行在操作系统的进程仍然可以互相影响

3. 大约1980年，虚拟内存地址空间技术出现，每个进程的内存空间被隔离开来

4. 大约1990年，in-process seperation也流行开来，最主要的做法是隔离解释器和被解释代码
例如，java解释器和java代码，python解释器和python代码

5. 大约2000年，浏览器的攻击手法丰富了起来，当时主要围绕三个浏览器特性进行攻击：
    Adobe Flash
    ActiveX
    Java Applets
通过这些特性，攻击者需要让受害者访问恶意网址，触发漏洞，就能控制受害者系统

6. 大约2010年，浏览器也出了很多的缓解措施。
比如直接关闭这些特性
像Adobe Flash、ActiveX、Java Applets都不允许被使用了
但是这出现了问题：攻击者把目光投向了浏览器的其他特性：
JS engine(JS解析器)漏洞、Media Codec(音频编解码器)漏洞，Imaging library(图像库)漏洞

有些有识之士提出的解决方案是不受信任的代码/数据应该存活在进程的zero-permissions状态下
大致方法是发起父子进程，父进程为高权限进程，子进程为不受信任进程（存放不受信任代码/数据），子进程每次需要运行权限操作，都要经过父进程同意,沙箱应运而生
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241020192813.png)<br>
**沙箱是一个非常强力的缓解措施**，它直接导致了：<br>
```
1. 需要一系列漏洞以利用沙盒进程
2. 需要另一系列漏洞来打破沙盒
```


## 2. chroot<br>
传统的沙盒就是`chroot`，`chroot`第一次出现在1979年的UNIX系统上，随后不久也出现在了BSD上。<br>
**chroot修改了 '/' 对于一个进程（包括其子进程）的含义**<br>
比如<br>
```c
chroot("/tmp/jail");
//会让进程认为 '/tmp/jail'目录（操作系统中的）就是自己的'/'目录 
```
所以`chroot`是一个事实上的`sandboxing utility(沙盒功能组件)`<br>
**值得注意的一点是，chroot并没有禁止系统调用，也没有其他隔离功能**<br>

### 2.1 chroot使用注意事项<br>
使用`chroot`时有很多点需要注意:<br>
```
1. chroot系统调用执行时需要privilege，一般情况下需要root权限，所以执行时若不是root用户，需要执行sudo
```
不使用root权限的代价如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241021225618.png)<br>

```
2. 被执行的命令或程序，需要在被限制的目录下
比如sudo chroot /tmp /bin/bash
这种情况下，在/tmp/bin目录中需要有bash文件
```
不在`tmp目录下放/bin/bash`实验结果如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241021225911.png)

```
3. 被执行程序，需要是静态编译好的，否则，需要把动态链接所需的所有库都放入jail当中
```
值得一提的是，如果动态库没有放好，也会报没有文件或目录的错误，如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241022213844.png)
如果你在想要的目录下放好了所有动态程序所需的依赖,效果如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241022214620.png)
当然，个人建议还是使用静态程序，比较安逸，这里推荐使用**busybox（集成了很多常见的unix命令，像ls cat这些都有）**，busybox官网[https://busybox.net/](https://busybox.net/)，可以直接在上面下载编译好的程序使用<br>
使用busybox的图如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241030194122.png)

### 2.2 chroot作用<br>
```
chroot("/tmp/jail") 有2个作用:

1. 对于这个进程而言，改变 "/"的含义，使其为 "/tmp/jail"

2. 然后下面的一切： "/flag" 变成 "/tmp/jail/flag",对于这个进程而言，"/tmp/jail/.." 指向 "/tmp/jail"
```
chroot是有缺点的<br>
```
chroot("/tmp/jail") 不能:
1. 关闭原本已经开启的指向其他目录（非jail目录）的文件描述符等系统资源。
2. cd (chdir()) 来进入jail，需要显示得调用chdir("/");才行
3. 做其他事情！
```

### 2.3 chroot陷阱<br>
先前提到，`chroot`不会对原本已经开启的文件描述符等系统资源做限制。<br>
```
与  open 和 execve类似, Linux 还有 openat 和 execveat 的系统调用:
  int open(char *pathname, int flags);
  int openat(int dirfd, char *pathname, int flags);
  int execve(char *pathname, char **argv, char **envp);
  int execveat(int dirfd, char *pathname, char **argv, char **envp, int flags);
dirfd 能表示为任何一个打开着的目录文件描述符, 或者是特殊值 AT_FDCWD(在linux代表的是当前工作目录) (注意: chroot()
不会改变当前工作目录)!
```
另外，内核并不会记得程序已经在一个jail当中，也就是说，你可以通过在使用一次`chroot("/")`来跳出jail<br>

### 2.4 chroot安全性讨论<br>
当然`chroot`不安全，首先，`euid`为0(即具备root或suid)的程序可以随时跳出jail,除非`chroot`被禁用了<br>
另外，`chroot`也没有对做`PID network IPC`的隔离，比如：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241031214105.png)
在知道了PID后，你可以开启其他终端来中断jail的运行。

## 3. seccomp<br>
现代的sandboxing会严格的限制进程的系统调用；这主要通过一个内核级的sandboxing机制——`seccomp`来实现<br>
`seccomp`机制能够允许开发者写下复杂的规则来完成以下事情:<br>
```
1. 允许进程使用system call
2. 禁止进程使用system call
3. 筛选允许和不允许的系统调用
```

**另外有一点，当前进程的`seccomp`是能够被其子进程继承的！**<br>

### 3.1 seccomp示例<br>
```C
//gcc seccomp.c -o seccomp -lseccomp
#include <sys/sendfile.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <assert.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <time.h>
#include <fcntl.h>

int main(){
    scmp_filter_ctx ctx;
    ctx = seccomp_init(SCMP_ACT_ALLOW);
    seccomp_rule_add(ctx,SCMP_ACT_KILL,SCMP_SYS(read),0);
    seccomp_load(ctx);
    execl("/bin/cat","cat","/flag",0);
}

```
实际上，作为该进程的子进程`cat`无法读取flag！<br>
用`strace`跟踪系统调用，结果如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241101202842.png)

### 3.2 seccomp工作原理<br>
seccomp实际上是通过`prctl`系统调用来实现控制的，而`prctl`系统调用又依赖于`eBPF(extended Berkeley Packet Filters)`,eBPF是运行于内核态的一个”可证明安全“的虚拟机，它能在内核中插入用户想要实现的功能：<br>
```
1. 通常用于网络包过滤（比如iptables）
2. 可用于实现系统调用跟踪和限制！
```
bpf详情可参考[https://github.com/iovisor/bcc](https://github.com/iovisor/bcc)<br>
