---
layout: post
tags: [pwn]
title: "sandboxing —— namespace"
author: wsxk
date: 2024-11-17
comments: true
---

- [1. 什么是 namespaces](#1-什么是-namespaces)
  - [1.1 namespaces 系统api](#11-namespaces-系统api)
- [namespace 和 seccomp 的差异和关联](#namespace-和-seccomp-的差异和关联)
- [PS：docker的隔离原理](#psdocker的隔离原理)
- [附录: namespaces系统调用api使用方法](#附录-namespaces系统调用api使用方法)


## 1. 什么是 namespaces<br>
`namespaces`的简介可以通过linux命令 `man namespaces`来了解<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241117192240.png)
总而言之，`namespaces`是linux kernel提供的用来隔离进程可用内核资源的机制。通过`namespaces`可以让进程只看到跟自己相关的一部分内核资源。<br>
从上述的`man`可以得知，当前`namespaces`只允许隔离7种类型的linux内核资源<br>
```
1. cgroup: 

2. IPC: 进程间通信的三种方式：共享内存、消息队列、信号量。但是容器里的进程间通
信，对于宿主机而言，其实是具有相同的 PID namespace的进程间通信，这里创建的是

3. Network:

4. Mount:

5. PID:

6. User:

7. UTS：提供主机名和域名的隔离，这样一个容器就可以拥有独立的主机名和域名，能够
作为网络中的一个独立节点，而非宿主机上的进程
```

### 1.1 namespaces 系统api<br>
需要用到的API有三个<br>
```c
// 创建一个新的进程的同时，创建新的namespaces,且新进程会被附加到新namespaces中
// 注意：父进程仍然在原来的namespaces中
int clone(int (*fn)(void *), void *child_stack,
       int flags, void *arg, ...
       /* pid_t *ptid, void *newtls, pid_t *ctid */ );

/*
fn：指的是新进程运行的函数指针
child_stack: 子进程使用的栈空间指针
flags ：使用哪些CLONE_*标志位，不同标准位代表不同类型的namespaces
arg ： 传入用户的参数
*/
```

```c
//创建一个新的命名空间，并将当前进程附加到新的namespaces中，flags与clone的flags一致
int unshare(int flags);
```

```c
//将当前进程附加到已有的namespaces中
int setns(int fd, int nstype);

/*
fd: 我们要加入的namespaces的文件描述符，通常是/proc/[pid]/ns下面的文件描述符
nstype： 让调用者检查fd指向的文件描述符是否符合实际要求，填0则不检查
*/
```
文件描述符查看的例子：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241119000111.png)


## namespace 和 seccomp 的差异和关联<br>
`namespace`用于限制进程可调用的系统资源，`seccomp`用于限制进程可以执行的系统调用；一定要说的话**seccomp的优先级大于namespace,毕竟namespace的使用依赖于执行系统调用（system calls）**<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## PS：docker的隔离原理<br>
尚未证实的说法，据说`docker = chroot + namespace + seccomp`<br>
话又说回来，即使是这么简单的思路，实践起来也很困难，不然世上也就不会仅docker一家独大了。<br>
**有的时候理解原理思路 跟 落地实践 是两码事，纸上得来终觉浅，须知此事要躬行，古人诚不欺我**<br>
要想学好网络安全，实践是必不可少的.以后还是要专注于实践。<br>

## 附录: namespaces系统调用api使用方法<br>
可以参考[https://blog.csdn.net/huchao_lingo/article/details/140448672](https://blog.csdn.net/huchao_lingo/article/details/140448672)文章，写的挺好的，yysy<br>
里面还有针对namespace各个参数的案例代码，可以运行感受一下<br>
