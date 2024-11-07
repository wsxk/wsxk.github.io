---
layout: post
title: "seccomp学习"
date:   2022-6-5
tags: [linux]
comments: true
author: wsxk
---

- [介绍](#介绍)
- [安装](#安装)
  - [1.正常64位安装](#1正常64位安装)
  - [2.32位安装](#232位安装)
- [seccomp使用](#seccomp使用)
  - [1. seccomp初始化](#1-seccomp初始化)
  - [2. 添加规则](#2-添加规则)
  - [3.加载和卸载](#3加载和卸载)
  - [4.实例](#4实例)
- [编译](#编译)
- [父进程设置的规则同样适用于子进程](#父进程设置的规则同样适用于子进程)
- [reference](#reference)

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


# 介绍

为了保护系统安全，用户层的应用程序使用计算机资源需要通过系统调用，Linux即可以通过C库函数进行系统调用，但是并不是所有的系统调用都会被用到，其中有些敏感的系统调用可能会被误用，比如pwn题中经常通过system、execve 来etshell，为了防止这种情况发生沙盒应运而生，通过沙盒可以限制系统调用的使用极大提高系统安全性。

seccomp(security computing mode)就是著名的沙盒之一。它从Linux 2.6.10之后引入到kernel中，我们可以使用它来限制系统调用

ctf中著名的orw类型的题目，就是基于seccomp来设置题目的。

# 安装

## 1.正常64位安装

早期的seccomp需要prctl系统调用来实现作用，后来为了方便使用封装了libseccomp库直接实现seccomp

这里直接介绍使用libseccomp库的，方便快捷，控制粒度也会更细一点。

    apt install libseccomp-dev libseccomp2 seccomp

安装好后，在c代码里使用

    #include <seccomp.h>

即可。

## 2.32位安装

默认情况下给你安装的libseccomp都是64位的（如果你的机子是64位的话），而且你会发现，你用apt search seccomp后，得到的结果都是64位的，似乎没有32位的，你怎么办？

    sudo apt-get install libseccomp-dev:i386

# seccomp使用

## 1. seccomp初始化

scmp_filter_ctx是过滤器的结构体类型

seccomp_init对结构体进行初始化，若参数为SCMP_ACT_ALLOW，则没有匹配到规则的系统调用将被默认允许，过滤为黑名单模式；若为SCMP_ACT_KILL，则为白名单模式，即没有匹配到规则的系统调用都会杀死进程，默认不允许所有的syscall。

一般初始化为

    scmp_filter_ctx ctx;// Init the filter
    ctx = seccomp_init(SCMP_ACT_ALLOW);

## 2. 添加规则

seccomp_rule_add是添加一条规则，其函数原型如下：

    int seccomp_rule_add(scmp_filter_ctx ctx, uint32_t action, int syscall, unsigned int arg_cnt, ...);

ctx是初始化的结构体，action就是行为，即允许运行或不允许运行等待，syscall是系统调用号，arg_cnt是表明要限制的参数个数，如果你不限制参数，就填0.

    seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);即禁用execve，不管其参数如何。

    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(dup2), 2, SCMP_A0(SCMP_CMP_EQ, 1), SCMP_A1(SCMP_CMP_EQ, 2));即当调用dup2函数时，只有前两个参数为1和2时，才允许调用

## 3.加载和卸载

添加好规则后，使用

    seccomp_load(ctx)

若想删除，可以使用

    seccomp_release(ctx);

## 4.实例

```c
    #include <unistd.h>
    #include <seccomp.h>
    
    int main(void){
        scmp_filter_ctx ctx;// Init the filter
        ctx = seccomp_init(SCMP_ACT_ALLOW);
        seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);
        seccomp_load(ctx); //加载
    
        char * str = "/bin/sh";
        write(1,"i will give you a shell\n",24);
        syscall(59,str,NULL,NULL);//execve
        return 0;
    }
```

# 编译

    gcc test.c -o test -l seccomp

# 父进程设置的规则同样适用于子进程

如标题。

# reference

[https://blog.51cto.com/u_15127593/3259635](https://blog.51cto.com/u_15127593/3259635)

[https://blog.csdn.net/qq_44846324/article/details/121731640](https://blog.csdn.net/qq_44846324/article/details/121731640)