---
layout: post
tags: [pwn]
title: "sandboxing —— namespaces"
author: wsxk
date: 2024-11-17
comments: true
---

- [1. 什么是 namespaces](#1-什么是-namespaces)
- [2. 运用namespaces构建container](#2-运用namespaces构建container)
  - [2.1 namespaces使用前提](#21-namespaces使用前提)
  - [2.2 运用linux cmds构建container](#22-运用linux-cmds构建container)
- [3. namespaces 和 seccomp 的差异和关联](#3-namespaces-和-seccomp-的差异和关联)
- [4. PS：docker的隔离原理](#4-psdocker的隔离原理)
- [5. 附录: C/C++: namespaces API使用方法](#5-附录-cc-namespaces-api使用方法)
  - [5.1 namespaces 系统api](#51-namespaces-系统api)
  - [5.2 A example](#52-a-example)


## 1. 什么是 namespaces<br>
`namespaces`的简介可以通过linux命令 `man namespaces`来了解<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241117192240.png)
总而言之，`namespaces`是linux kernel提供的用来隔离进程可用内核资源的机制。通过`namespaces`可以让进程只看到跟自己相关的一部分内核资源。<br>
从上述的`man`可以得知，当前`namespaces`只允许隔离7种类型的linux内核资源<br>
```
1. UTS：提供主机名和域名的隔离，这样一个容器就可以拥有独立的主机名和域名，能够
作为网络中的一个独立节点，而非宿主机上的进程

2. IPC: 进程间通信的三种方式：共享内存、消息队列、信号量。这让容器内无法见到宿
主机的进程间通信的状态。（可通过ipcmk -Q创建队列，ipcs -q查看队列 ipcrm -Q key删除队列）
反过来，宿主机也无法看到容器内新建的队列

3. PID: 创建独立的PID namespace，同一个进程在不同的namespace下可以有不同的pid。
另外，因为linux内核为所有的PID namespace维护了一个树状结构，树顶就是linux内
核默认创建的root namespace，我们创建的新PID namespace被成为child namespace
因此，pid namespace是有等级的，所属父节点可以看到子节点的进程，并通过发送信号
等方式影响子节点
换句话说，通常情况下容器内创建的进程，在宿主机是可以观测到并杀死的，反之不行。
此时如果ps -aux还是可以看到所有的进程，这说明隔离不完全（没有隔离/proc文件系统）

4. Mount:创建一个mount namespace，通过隔离文件系统挂载点来隔离文件系统，在
执行mount("proc","/proc","proc",0,NULL);后（对应参数的含义分别是
挂载的源为proc文件系统，挂载到/proc目录下，挂载的类型是proc）
ps -aux将无法看到其他进程的信息，退出该进程后需要sudo mount -t proc proc /proc进行复原

5. User:一个普通用户的进程通过clone()创建新的进程在新user namespace中可以拥
有不同的用户和用户组。这意味着一个进程在容器外属于一个没有特殊权限的普通用户，
但是它创建的容器进程却属于拥有所有权限的超级用户，这个技术为容器提供了极大的自由。

6. Network: 能够隔离网络设备，栈，端口，等等

7. cgroup:  能够限制被隔离的namespace中，使被隔离的namespace中的进程，对CPU
的使用率，内存使用量，磁盘IO速率，网络带宽等等进行限制！
```
namespaces用途广泛，我们应该如何使用呢？<br>

## 2. 运用namespaces构建container<br>
### 2.1 namespaces使用前提<br>
需要有`root`权限，毕竟需要隔离系统资源<br>

### 2.2 运用linux cmds构建container<br>
要想完整得运用`namespaces`构建容器，需要用到很多涉及linux kernel底层机制的命令，比如`mount`，正好总结一下这些命令背后的原理以及如何使用<br>


## 3. namespaces 和 seccomp 的差异和关联<br>
`namespace`用于限制进程可调用的系统资源，`seccomp`用于限制进程可以执行的系统调用；一定要说的话**seccomp的优先级大于namespace,毕竟namespace的使用依赖于执行系统调用（system calls）**<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## 4. PS：docker的隔离原理<br>
尚未证实的说法，据说`docker = chroot + namespace + seccomp`<br>
话又说回来，即使是这么简单的思路，实践起来也很困难，不然世上也就不会仅docker一家独大了。<br>
**有的时候理解原理思路 跟 落地实践 是两码事，纸上得来终觉浅，须知此事要躬行，古人诚不欺我**<br>
要想学好网络安全，实践是必不可少的.以后还是要专注于实践。<br>

## 5. 附录: C/C++: namespaces API使用方法<br>
### 5.1 namespaces 系统api<br>
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

### 5.2 A example<br>

可以参考[https://blog.csdn.net/huchao_lingo/article/details/140448672](https://blog.csdn.net/huchao_lingo/article/details/140448672)文章，写的挺好的，yysy<br>
里面还有针对namespace各个参数的案例代码，可以运行感受一下<br>
举个例子:<br>
```c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
 
#define STACK_SIZE (1024*1024)
 
static char child_stack[STACK_SIZE];
char* const child_args[] = {
	"/bin/bash",
	NULL
};
 
int child_main(void* args){
	printf("in child process!\n");
	sethostname("changed namespace", 12);
	execv(child_args[0], child_args);
	return 1;
}
 
int main(){
	printf("program begin: \n");
	int child_pid = clone(child_main, child_stack + STACK_SIZE, SIGCHLD|CLONE_NEWUTS, NULL);
	waitpid(child_pid, NULL, 0);
	printf("quit\n");
	return 0;
}
```

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241119221847.png)