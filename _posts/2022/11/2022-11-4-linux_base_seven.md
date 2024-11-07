---
layout: post
tags: [kernel_pwn]
title: "linux内核基础 七 userfaultfd"
author: wsxk
date: 2022-11-4
comments: true
---

- [userfaultfd简介](#userfaultfd简介)
- [userfaultfd流程](#userfaultfd流程)
- [userfaultfd样例](#userfaultfd样例)
  - [step 1： 创建fd](#step-1-创建fd)
  - [step 2：创建一个匿名映射](#step-2创建一个匿名映射)
  - [step 3: 注册 userfaultfd功能](#step-3-注册-userfaultfd功能)
  - [step 4： 创建 userfault monitor线程监听](#step-4-创建-userfault-monitor线程监听)
  - [完整demo](#完整demo)
- [userfaultfd的应用场景](#userfaultfd的应用场景)
  - [pre-copy](#pre-copy)
  - [post-copy](#post-copy)
- [番外 CRIU（checkpoint/restore in userspace）](#番外-criucheckpointrestore-in-userspace)
- [userfault的缺陷](#userfault的缺陷)
  - [弥补:sysctl\_unprivileged\_userfaultfd=0](#弥补sysctl_unprivileged_userfaultfd0)
- [references](#references)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## userfaultfd简介<br>
~~**一切，都是为了效率**~~<br>
`userfaultfd`是linux内核为用户态提供的一种机制，其可以让用户态程序来处理原本只能由内核来处理的缺页异常问题。提高了用户编程的灵活性<br>
**其实允许用户态处理内核态任务以提高性能的想法越来越普及**，比如DPDK（DATA plane development kit）它的注册绕过了linux原有的处理报文的繁琐流程（网卡收到报文发送信号给操作系统、操作系统获取报文进入内核协议栈处理、处理后将报文发送给相应的用户程序）,简化为 用户通过设备映射来直接与网卡通信，通过轮询来处理报文（利用mmap和修改系统调用）<br>
[https://www.elecfans.com/news/1238266.html](https://www.elecfans.com/news/1238266.html)<br>
有人可能会问，为什么内核不直接修改原有处理方式呢？就本人思考有以下2个主要原因:
> 1. 内核已经十分复杂了，直接改太麻烦了
> 2. 内核需要调度所有的任务，它没有权利给某个任务开小灶（绕过正常流程），这个权力是把握在用户手上的，应该由用户决定 当前任务是否能绕过内核

## userfaultfd流程<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221104165304.png)
该流程图的步骤如下:
> 1. 某个线程（Faulting thread） 读取了一个 mmap 得到的内存，但是内存未初始化，触发pagefault问题，产生信号
> 2. 该信号被内核转发给了userfaultfd
> 3. 内核时出问题线程进入休眠，向monitior（实际处理缺页问题的线程）发送消息
> 4. monitor监听到了该消息，做出相应处理后，通知userfaultfd
> 5. 内核唤醒出错线程。

## userfaultfd样例<br>
为了加深理解，写了下userfaultfd的demo来尝试。<br>
### step 1： 创建fd<br>
```c
    long user_fault_fd = syscall(__NR_userfaultfd,O_CLOEXEC|O_NONBLOCK);
    if(user_fault_fd<0){
        printf("create user_fault_fd error!\n");
        return;
    }
```
只能通过系统调用（syscall）直接获取一个fd其中
`O_CLOEXEC表示执行exec()时，子进程关闭该描述符`<br>
`O_NONBLOCK表示非阻塞`

### step 2：创建一个匿名映射<br>
```c
    page_size = sysconf(_SC_PAGE_SIZE);
    char* addr = mmap(NULL,page_size,PROT_READ|PROT_WRITE,MAP_PRIVATE|MAP_ANONYMOUS,-1,0);
    if(addr==MAP_FAILED){
        printf("error in mmap!\n");
        return ;
    }
```
先创建一个匿名的映射来做实验<br>

### step 3: 注册 userfaultfd功能<br>
```c
    struct uffdio_register uffdio_register;
    uffdio_register.range.start = (unsigned long) addr; //被监管的内存地址，是step2 创建的匿名内存区域
    uffdio_register.range.len = page_size;
    uffdio_register.mode = UFFDIO_REGISTER_MODE_MISSING;
    struct uffdio_api uffdio_api;
    uffdio_api.api = UFFD_API;
    uffdio_api.features = 0;
    ioctl(user_fault_fd, UFFDIO_API, &uffdio_api);
    ioctl(user_fault_fd, UFFDIO_REGISTER, &uffdio_register);
```

### step 4： 创建 userfault monitor线程监听<br>
```c
void * uffdio_handle_thread(void * user_fault_fd){
    static char * page = NULL;
    static struct uffd_msg msg;
    struct uffdio_copy uffdio_copy;
    long uffd = (long)user_fault_fd;
    if (page == NULL) 
    {
        page = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
                    MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
        printf("thread page addr:%p\n",page);
        if (page == MAP_FAILED){
            printf("error in mmap!\n");
            exit(0);
        }
    }
    memset(page,0x41,page_size);
    printf("thread test\n");

    for(;;){
        struct pollfd pollfd;
        pollfd.fd = uffd;
        pollfd.events=POLLIN;
        poll(&pollfd,1,-1);
        read(uffd,&msg,sizeof(msg));
        if(msg.event != UFFD_EVENT_PAGEFAULT){
            continue;
        }
        uffdio_copy.src = (unsigned long)page;
        uffdio_copy.dst = (unsigned long)msg.arg.pagefault.address & ~(page_size-1);
        uffdio_copy.len = page_size;
        uffdio_copy.mode = 0;
        uffdio_copy.copy = 0;
        ioctl(uffd,UFFDIO_COPY,&uffdio_copy);
    }
}
    int s = pthread_create(&uffdio_handler,NULL,uffdio_handle_thread,(void *)user_fault_fd);
```

### 完整demo<br>
```c
#include <sys/types.h>
#include <stdio.h>
#include <linux/userfaultfd.h>
#include <pthread.h>
#include <errno.h>
#include <unistd.h>
#include <stdlib.h>
#include <fcntl.h>
#include <signal.h>
#include <poll.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <sys/ioctl.h>
#include <poll.h>

static int page_size;

void * uffdio_handle_thread(void * user_fault_fd){
    static char * page = NULL;
    static struct uffd_msg msg;
    struct uffdio_copy uffdio_copy;
    long uffd = (long)user_fault_fd;
    if (page == NULL) 
    {
        page = mmap(NULL, page_size, PROT_READ | PROT_WRITE,
                    MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
        printf("thread page addr:%p\n",page);
        if (page == MAP_FAILED){
            printf("error in mmap!\n");
            exit(0);
        }
    }
    memset(page,0x41,page_size);
    printf("thread test\n");

    for(;;){
        struct pollfd pollfd;
        pollfd.fd = uffd;
        pollfd.events=POLLIN;
        poll(&pollfd,1,-1);
        read(uffd,&msg,sizeof(msg));
        if(msg.event != UFFD_EVENT_PAGEFAULT){
            continue;
        }
        uffdio_copy.src = (unsigned long)page;
        uffdio_copy.dst = (unsigned long)msg.arg.pagefault.address & ~(page_size-1);
        uffdio_copy.len = page_size;
        uffdio_copy.mode = 0;
        uffdio_copy.copy = 0;
        ioctl(uffd,UFFDIO_COPY,&uffdio_copy);
    }
}

int main(){
    
    long user_fault_fd = syscall(__NR_userfaultfd,O_CLOEXEC|O_NONBLOCK);
    if(user_fault_fd<0){
        printf("create user_fault_fd error!\n");
        return;
    }
    page_size = sysconf(_SC_PAGE_SIZE);
    char* addr = mmap(NULL,page_size,PROT_READ|PROT_WRITE,MAP_PRIVATE|MAP_ANONYMOUS,-1,0);
    if(addr==MAP_FAILED){
        printf("error in mmap!\n");
        return ;
    }
    struct uffdio_register uffdio_register;
    uffdio_register.range.start = (unsigned long) addr;
    uffdio_register.range.len = page_size;
    uffdio_register.mode = UFFDIO_REGISTER_MODE_MISSING;
    struct uffdio_api uffdio_api;
    uffdio_api.api = UFFD_API;
    uffdio_api.features = 0;
    ioctl(user_fault_fd, UFFDIO_API, &uffdio_api);
    ioctl(user_fault_fd, UFFDIO_REGISTER, &uffdio_register);
    printf("test main\n");
    pthread_t uffdio_handler;
    int s = pthread_create(&uffdio_handler,NULL,uffdio_handle_thread,(void *)user_fault_fd);
    if(s!=0){
        printf("error create thread\n");
        return ;
    }
    printf("main page addr:%p\n",addr);
    void * ptr = (void*) *(unsigned long long*) addr;
    printf("data:%p\n",ptr);

}
```

可以看看运行效果:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221104165031.png)
原来，main函数用mmap创建的空间实际上是没有分配到实际物理内存的（因为我们什么都没干），因此直接访问该内存会出发异常，是获取不到内容的。<br>
**添加了userfaultfd后，monitor可以监听到该pagefault，并利用我们自己的处理，将内存上的内容设定为了全是'A'**<br>

## userfaultfd的应用场景<br>
userfaultfd主要应用在虚拟机的`live migration`中<br>
`live migration`主要有2种方式，`pre-copy`和`post-copy`<br>

### pre-copy<br>
pre-copy的思路是，从源到端进行多轮迭代拷贝。**第一次拷贝先从源拷贝所有内容，第二次拷贝在第一次拷贝的过程中发生更改的数据，第三次拷贝第二次拷贝过程中发生更改的数据。。。。此次类推，直到某一次更改的数据量小于一个阈值，这时候暂停源虚拟机将剩余的数据拷贝到目的端虚拟机，流程结束**
<br>

### post-copy<br>
post-copy就利用到了`userfaultfd`机制。<br>
post-copy跟pre-copy不同，它要求目的端虚拟机已经有了和源端虚拟机一样的镜像<br>
随后暂停发生在源端的最开始，目的端只需要拷贝源端虚拟机运行状态相关的少量数据（CPU状态、寄存器、无页表内存等等） 即可，然后运行程序时的内存页也是按需调度（需要时再从源那边拷贝相应内存页的内容即可）。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221104172445.png)
<br>

同样利用了 这种技术的还有 CRIU<br>

## 番外 CRIU（checkpoint/restore in userspace）<br>
CRIU技术主要用于冻结当前虚拟机正在运行的进程，然后可以复制到其他的虚拟机上继续运行（运行状态和原本运行时没有差别，包括内存分布情况）<br>
它和post-copy技术原理十分类似，在page server在本地/远程都有相应的实现逻辑<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221104173108.png)

## userfault的缺陷<br>
一般情况下，优化了性能地同时会带来安全隐患<br>
userfault机制提供了内核竞争的稳定利用<br>
(通过上述的机制描述，我们发现，我们可以向内核传入未映射物理内存的区域时会触发缺页异常使发生该问题的线程进入睡眠，因而可以控制内核竞争的执行流程，大大提高利用率)<br>

### 弥补:sysctl_unprivileged_userfaultfd=0<br>
新版的linux内核（5.11及往上）都默认设置了sysctl_unprivileged_userfaultfd = 0，这使得普通权限的用户执行userfault失败（权限不足）<br>

## references<br>
[https://ctf-wiki.org/pwn/linux/kernel-mode/exploitation/userfaultfd/](https://ctf-wiki.org/pwn/linux/kernel-mode/exploitation/userfaultfd/)<br>
[http://brieflyx.me/2020/linux-tools/userfaultfd-internals/](http://brieflyx.me/2020/linux-tools/userfaultfd-internals/)<br>
[https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#userfaultfd](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#userfaultfd)<br>
[https://blog.csdn.net/qq_43375973/article/details/117387384](https://blog.csdn.net/qq_43375973/article/details/117387384)<br>
[https://zhuanlan.zhihu.com/p/570868104](https://zhuanlan.zhihu.com/p/570868104)<br>
[https://www.elecfans.com/news/1238266.html](https://www.elecfans.com/news/1238266.html)<br>