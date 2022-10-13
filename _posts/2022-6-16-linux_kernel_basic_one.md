---
layout : post
title: "linux内核基础 一"
tags: [kernel_pwn]
date: 2022-6-16
comments: true
author: wsxk
---

`更新于2022-10-12`<br>
- [什么是内核<br>](#什么是内核)
- [内核架构<br>](#内核架构)
  - [宏内核(Monolithic Kernel)<br>](#宏内核monolithic-kernel)
  - [微内核（Micro Kernel）<br>](#微内核micro-kernel)
- [linux系统启动过程<br>](#linux系统启动过程)
  - [1.加载BIOS（Basic I/O System）<br>](#1加载biosbasic-io-system)
  - [2.读取MBR（Master Boot Record,主引导记录）<br>](#2读取mbrmaster-boot-record主引导记录)
  - [3.(GNU GRUB)引导<br>](#3gnu-grub引导)
  - [4.加载kernel<br>](#4加载kernel)
  - [5.启动守护进程init<br>](#5启动守护进程init)
- [kernel部分原理<br>](#kernel部分原理)
  - [1.分级保护域（hierarchical protection domains,简称rings）<br>](#1分级保护域hierarchical-protection-domains简称rings)
    - [用户态和内核态<br>](#用户态和内核态)
    - [用户空间和内核空间<br>](#用户空间和内核空间)
    - [用户态->内核态<br>](#用户态-内核态)
    - [内核态->用户态<br>](#内核态-用户态)
  - [2.进程管理<br>](#2进程管理)
    - [进程描述符（process descriptor）<br>](#进程描述符process-descriptor)
    - [提权<br>](#提权)
  - [3.I/O<br>](#3io)
    - [进程文件系统(process file system，procfs)<br>](#进程文件系统process-file-systemprocfs)
    - [文件描述符(file descriptor,fd)<br>](#文件描述符file-descriptorfd)
  - [4.Loadable Kernel Modules（LKMs）<br>](#4loadable-kernel-moduleslkms)
  - [5.内核内存管理<br>](#5内核内存管理)
    - [buddy system<br>](#buddy-system)
      - [page<br>](#page)
      - [pageblock<br>](#pageblock)
      - [page order<br>](#page-order)
      - [迁移类型<br>](#迁移类型)
      - [buddy system分配过程<br>](#buddy-system分配过程)
    - [slab alloctor<br>](#slab-alloctor)
      - [slab allocator的版本<br>](#slab-allocator的版本)
      - [slub数据结构<br>](#slub数据结构)
      - [slub分配过程<br>](#slub分配过程)
      - [slub释放过程<br>](#slub释放过程)
    - [mmap、brk怎么申请内存<br>](#mmapbrk怎么申请内存)
  - [6.kernel保护机制<br>](#6kernel保护机制)
    - [1.KASLR(kernel address space layout randomize,内核地址空间随机化)<br>](#1kaslrkernel-address-space-layout-randomize内核地址空间随机化)
    - [2.FGKASLR<br>](#2fgkaslr)
    - [3.STACK PROTECTOR<br>](#3stack-protector)
    - [4.SMAP/SMEP（Supervisor Memory Access/Execution Prevention）<br>](#4smapsmepsupervisor-memory-accessexecution-prevention)
    - [5.KPTI(kernel page-table isolation,内核页表隔离)<br>](#5kptikernel-page-table-isolation内核页表隔离)
    - [6.heap保护<br>](#6heap保护)
      - [①Hardened Usercopy<br>](#hardened-usercopy)
      - [②Hardened freelist<br>](#hardened-freelist)
      - [③Random freelist<br>](#random-freelist)
- [reference<br>](#reference)

# 什么是内核<br>
***操作系统（Operation System）*** 本质上也是一种软件，可以看作是普通应用程式与硬件之间的一层中间层，其主要作用便是调度系统资源、控制IO设备、操作网络与文件系统等，并为上层应用提供便捷、抽象的应用接口<br>
而 ***内核（kernel）*** 是操作系统最重要的一部分。<br>
>通常来说，操作系统= 内核+文件系统（file system）
>> 文件系统简单理解就是用来管理存放在硬盘上的文件和目录的。
>> kernel提供了除文件系统以外的其他功能:
>>1.控制并与硬件进行交互
>>2.提供应用程式运行环境
>>3.调度系统资源
>>>  I/O，权限控制，系统调用，进程管理，内存管理等多项功能都可以归结到以上三点中


有一张图可以直观地看出内核的在计算机体系结构中的位置
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220616162349.png)

# 内核架构<br>
传统的内核架构有两种: ***宏内核和微内核*** <br>
## 宏内核(Monolithic Kernel)<br>
宏内核顾名思义，它其实是一个完整的可执行的二进制程序，在内核态以监管者模式（Supervisor Mode）来运行。<br>
宏内核将所有提供的功能都打包了起来，并向上层程序提供API接口。

## 微内核（Micro Kernel）<br>
微内核，大部分的系统服务（如文件管理等）都被剥离于内核之外，内核仅仅提供最为基本的一些功能：底层的寻址空间管理、线程管理、进程间通信等。
> 虽然有宏内核和微内核各有优势，宏内核因为所有功能集中在一起，效率比较高（但是维护困难，不易扩展），微内核因为模块化，所以维护方便，设计难度大大降低（但是因为不同模块的通信问题导致效率降低）。当然了历史告诉我们，出现了两个极端的设计理念后，最理想的理念往往是取长补短，于是出现了混合内核（hybrid kernel）。
>>举个例子，linux的内核虽然整体架构是宏内核，但是为了提高可扩展性和可维护性，也允许用户自己编写模块并在内核中部署（可装载内核模块（Loadable Kernel Modules，简称LKMs）
>>微软虽然宣称自己是微内核，但是它里面也有一些重要的功能必须集成到内核上。 

# linux系统启动过程<br>
根据我自己学习内核基础知识的过程，我觉得大家可能会对linux系统的引导过程产生异或，因此在真正讲述基础知识前，我会先解释一下linux的启动过程。

## 1.加载BIOS（Basic I/O System）<br>
当你开启计算机电源，首先加载的是基本输入输出系统（Basic Input Output System ），就是BIOS<br>
>bios程序一般放在主板ROM（Read Only Memory)中，关机或掉电也不会消失。

BIOS中包含了CPU的相关信息、设备启动顺序信息、硬盘信息、内存信息、时钟信息、PnP特性等等。在此之后，计算机就知道应该去读取哪个硬件设备了

## 2.读取MBR（Master Boot Record,主引导记录）<br>
读取硬盘上磁道的第一个扇区（MBR），它的大小是512字节.<br>
前446字节存放的就是grub程序的一部分、后64字节是硬盘分区表，里面存放了预启动信息、分区表信息。<br>
系统找到BIOS所指定的硬盘的MBR后，就会将其复制到0×7c00地址所在的物理内存中。其实被复制到物理内存的内容就是Boot Loader，而具体到你的电脑，那就是lilo或者grub了

## 3.(GNU GRUB)引导<br>
GRUB是boot loader的一种<br>
GRUB是多启动规范的实现，它允许用户可以在计算机内同时拥有多个操作系统，并在计算机启动时选择希望运行的操作系统

## 4.加载kernel<br>
根据grub指定的内核映像的路径，来加载内核映像<br>
系统将解压后的内核放置在内存之中，并调用start_kernel()函数来启动一系列的初始化函数并初始化各种设备，完成Linux核心环境的建立。至此，Linux内核已经建立起来了，基于Linux的程序应该可以正常运行了。

## 5.启动守护进程init<br>
init(sbin/init)是linux的第一个进程，这个进程读取相应的配置文件并启动一系列进程，这个进程的PID为1，所有的进程都由它衍生，都是它的子进程.<br>
这个进程会读取/etc/inittab文件，/etc/inittab文件的作用是设定Linux的运行等级以及执行项(比如下文出现的rc.sysinit和rc*.d,tty)来启动对应的程序<br>
>运行等级如下。
>•0：关机模式<br>
>•1：单用户模式<br>
>•2：无网络支持的多用户模式<br>
>•3：字符界面多用户模式<br>
>•4：保留，未使用模式<br>
>•5：图像界面多用户模式<br>
>•6：重新引导系统，重启模式

<br>
随后，init进程会fork出子进程执行/etc/rc.d/rc.sysinit（inittab文件告诉init要执行它）

>这个脚本对系统进行一系列的初始化，包括时钟、键盘、磁盘、文件系统等初始化

然后init执行启动层级对应脚本（rc*.d）,在/etc/inittab配置如下
>l0:0:wait:/etc/rc.d/rc 0<br>
>l1:1:wait:/etc/rc.d/rc 1<br>
>l2:2:wait:/etc/rc.d/rc 2<br>
>l3:3:wait:/etc/rc.d/rc 3<br>
>l4:4:wait:/etc/rc.d/rc 4<br>
>l5:5:wait:/etc/rc.d/rc 5<br>
>l6:6:wait:/etc/rc.d/rc 6<br>

rc执行完毕之后，系统环境已经设置完成，各种服务进程也已经启动。init开始启动终端程序。inittab文件中执行项通常如下

>1:2345:respawn:/sbin/mingetty tty1<br>
>2:2345:respawn:/sbin/mingetty tty2<br>
>3:2345:respawn:/sbin/mingetty tty3<br>
>4:2345:respawn:/sbin/mingetty tty4<br>
>5:2345:respawn:/sbin/mingetty tty5<br>
>6:2345:respawn:/sbin/mingetty tty6<br>

# kernel部分原理<br>
## 1.分级保护域（hierarchical protection domains,简称rings）<br>
分级保护域是一种将计算机不同的资源划分成不同权限的模型。<br>
在一些硬件或者微代码级别上提供不同特权态模式的 CPU 架构上，保护环通常都是硬件强制的。Rings是从最高特权级（通常被叫作0级）到最低特权级（通常对应最大的数字）排列的<br>
内层ring可以任意调用外层ring的资源，内层ring到外层ring可调用资源依次递减。<br>
通常情况下，内核运行在r0级，用户程序运行在r3级（其实中间2级基本上没什么人用）<br>
r3级其实没什么用，基本上啥都不能干（没权限），但是我们的用户程序也需要进行 访问文件，连接网络，与其他进程通信的操作，这时候应该怎么办？<br>
聪明的内核设计者们提出了一个聪明的办法：当用户程序(r3)需要访问系统资源时，会向操作系统发送请求，操作系统（r0）会帮你完成你的请求，然后返回用户进程(r3)。这里涉及了r3-r0-r3的切换，接下来会具体讲解这个

### 用户态和内核态<br>
用户态其实相当于r3，内核态相当于r0。

### 用户空间和内核空间<br>
为了确保操作系统的安全稳定运行，操作系统启动后，将会开启保护模式：将内存分为内核空间（内核对应进程所在内存空间，存储内核代码）和用户空间（存储用户代码），进行内存隔离<br>
处于用户态的程序只能访问用户空间，而处于内核态的程序可以访问用户空间和内核空间<br>

### 用户态->内核态<br>
用户态到内核态的切换，总的来说，有以下几种方式<br>
1.系统调用（软件中断）<br>
>这是用户态进程主动要求切换到内核态的一种方式，用户态进程通过系统调用申请使用操作系统提供的服务程序完成工作，比如fork()实际上就是执行了一个创建新进程的系统调用。而系统调用的机制其核心还是使用了操作系统为用户特别开放的一个中断来实现，例如Linux的int 80h中断

2.异常(出错（fault）和陷阱（trap）)<br>
>当CPU在执行运行在用户态下的程序时，发生了某些事先不可知的异常，这时会触发由当前运行进程切换到处理此异常的内核相关程序中，也就转到了内核态。异常分为出错和陷入两种。
>>出错（fault）保存的EIP指向触发异常的那条指令；而陷入（trap）保存的EIP指向触发异常的那条指令的下一条指令。因此，当从异常返回时，出错（fault）会重新执行那条指令；而陷入（trap）就不会重新执行。这一点实际上也是相当重要的，比如我们熟悉的缺页异常（page fault），由于是fault，所以当缺页异常处理完成之后，还会去尝试重新执行那条触发异常的指令（那时多半情况是不再缺页）。上文中提到的系统调用，其实属于trap的一种，int 3也是trap的一种，调试器原理之一就是int 3了。

3.外围设备的硬件中断（硬中断和软中断）<br>
>当外围设备完成用户请求的操作后，会向CPU发出相应的中断信号，这时CPU会暂停执行下一条即将要执行的指令转而去执行与中断信号对应的处理程序，如果先前执行的指令是用户态下的程序，那么这个转换的过程自然也就发生了由用户态到内核态的切换。比如硬盘读写操作完成，系统会切换到硬盘读写的中断处理程序中执行后续操作等。

回到具体的，当操作系统收到了用户态的请求时，会发生以下情况:

1.切换GS寄存器
>通过 swapgs 切换 GS 段寄存器，将 GS 寄存器值和一个特定位置的值进行交换，目的是保存 GS 值，同时将该位置的值作为内核执行时的 GS 值使用

2.保存用户态栈信息。
>将当前栈顶（用户空间栈顶）记录在 CPU 独占变量区域里，将 CPU 独占区域里记录的内核栈顶放入 rsp/esp

3.保存用户态寄存器信息.
>通过 push 保存各寄存器值到栈上，以便后续“着陆”回用户态

4.控制权转交内核，执行系统调用
>在这里用到一个全局函数表sys_call_table，其中保存着系统调用的函数指针

### 内核态->用户态<br>
内核态到用户态，只需要再执行一次swapgs切换寄存器，然后使用sysretq或者iretq恢复到用户空间即可.

## 2.进程管理<br>
### 进程描述符（process descriptor）<br>
在内核中，使用 ***task_struct*** 结构体来对每个进程进行管理,其结构如图所示<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220616194722.png)
这些大概知道一下内容就行，我们主要关注的也不是这个。<br>
在一个 ***task_struct*** 结构体中，有一部分声明：
``` c
/* Process credentials: */

/* Tracer's credentials at attach: */
const struct cred __rcu        *ptracer_cred;

/* Objective and real subjective task credentials (COW): */
const struct cred __rcu        *real_cred;

/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu        *cred;
```

这是进程权限凭证相关的部分
>ptracer_cred:使用ptrace系统调用跟踪该进程的上级进程的cred（gdb调试便是使用了这个系统调用，常见的反调试机制的原理便是提前占用了这个位置
>real_cred: ***客体凭证（objective cred）*** ,一个进程刚刚启动时的权限
>cred: ***主体凭证（subjective cred）*** ，该进程的有效cred，这才是真正标志进程权限的凭证。

现在仔细看看cred结构体的源码:
```c
struct cred {
    atomic_t    usage;
#ifdef CONFIG_DEBUG_CREDENTIALS
    atomic_t    subscribers;    /* number of processes subscribed */
    void        *put_addr;
    unsigned    magic;
#define CRED_MAGIC    0x43736564
#define CRED_MAGIC_DEAD    0x44656144
#endif
    kuid_t        uid;        /* real UID of the task */
    kgid_t        gid;        /* real GID of the task */
    kuid_t        suid;        /* saved UID of the task */
    kgid_t        sgid;        /* saved GID of the task */
    kuid_t        euid;        /* effective UID of the task */
    kgid_t        egid;        /* effective GID of the task */
    kuid_t        fsuid;        /* UID for VFS ops */
    kgid_t        fsgid;        /* GID for VFS ops */
    unsigned    securebits;    /* SUID-less security management */
    kernel_cap_t    cap_inheritable; /* caps our children can inherit */
    kernel_cap_t    cap_permitted;    /* caps we're permitted */
    kernel_cap_t    cap_effective;    /* caps we can actually use */
    kernel_cap_t    cap_bset;    /* capability bounding set */
    kernel_cap_t    cap_ambient;    /* Ambient capability set */
#ifdef CONFIG_KEYS
    unsigned char    jit_keyring;    /* default keyring to attach requested
                     * keys to */
    struct key    *session_keyring; /* keyring inherited over fork */
    struct key    *process_keyring; /* keyring private to this process */
    struct key    *thread_keyring; /* keyring private to this thread */
    struct key    *request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
    void        *security;    /* subjective LSM security */
#endif
    struct user_struct *user;    /* real user ID subscription */
    struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
    struct group_info *group_info;    /* supplementary groups for euid/fsgid */
    /* RCU deletion */
    union {
        int non_rcu;            /* Can we skip RCU deletion? */
        struct rcu_head    rcu;        /* RCU deletion hook */
    };
} __randomize_layout;
```
其中
> 1. real UID:标识一个进程启动时的用户ID（你是root启动进程，它就是root，你是普通用户它就是普通用户
> 2. saved UID:标识一个进程最初的有效用户ID
> 3. effective UID: 进程的真正权限
> 4. UID for VFS ops：标识一个进程创建文件时进行标识的用户ID

real GID,saved GID,effective GID,GID for VFS ops与上面的类似。

### 提权<br>
一个进程的权限是由位于内核空间的cred结构体进行管理的，只要改变一个进程的cred结构体，就能改变其执行权限<br>
内核提供了修改进程cred权限的函数

>struct cred* prepare_kernel_cred(struct task_struct* daemon)<br>
>该函数用以拷贝一个进程的cred结构体，并返回一个新的cred结构体，需要注意的是daemon参数应为有效的进程描述符地址或NULL

>int commit_creds(struct cred *new)<br>
>该函数用以将一个新的cred结构体应用到进程

prepare_kernel_cred有如下代码:
```c
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
    const struct cred *old;
    struct cred *new;

    new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
    if (!new)
        return NULL;

    kdebug("prepare_kernel_cred() alloc %p", new);

    if (daemon)
        old = get_task_cred(daemon);
    else
        old = get_cred(&init_cred);
...
```
当传入的参数为null时，会用init进程的cred来创建cred，init之前提到了，是linux系统的第一个进程，妥妥的root权限。

因此，在内核态执行 ***commit_creds(prepare_kernel_cred(NULL))*** 函数再返回到用户态，此时getshell就能拿到root权限。

## 3.I/O<br>
linux的设计思想之一是 ***万物皆文件*** <br>
设备，目录，文件，磁盘，管道，套接字....都被抽象成了文件<br>
> 所有读取操作都可以通过read进行
> 所有写操作都可以通过write进行

### 进程文件系统(process file system，procfs)<br>
进程文件系统是用来描述一个进程所打开的文件描述符、堆栈内存布局、环境变量等等<br>
procfs是一个伪文件系统，不会占用系统存储空间，通常挂载在/proc目录下。<br>
通常情况下，当一个进程在执行其间，其procfs会挂载在/proc/pid(进程pid)下。<br>

### 文件描述符(file descriptor,fd)<br>
进程通过文件描述符来完成对文件的操作和访问，fd在形式上是一个非负整数，其实是一个索引。<br>
每个进程都有独立的一个 ***文件描述符表*** ，存放着进程打开的文件索引。进程每打开(open)一个文件，内核会向进程返回一个文件描述符<br>
kernel中有一个共有的文件表，由所有进程共享。<br>
> 默认情况下，每个进程在启动时都会开启3个标准文件描述符，对应3个标准输入输出流。
> stdin 0
> stdout 1
> stderr 2
> linux提供了ioctl(int fd,unsigned long request,...)来提供对文件的访问。
> fd是文件描述符
> request是请求码
> ...是其余参数
> 对于一个提供了ioctl通信方式的设备而言，我们可以通过其文件描述符、使用不同的请求码及其他请求参数通过ioctl系统调用完成不同的对设备的I/O操作

## 4.Loadable Kernel Modules（LKMs）<br>
因为linux是宏内核架构，过于臃肿，有很多服务其实你根本用不上，这些服务直接装进内核太浪费资源了，但是你又不能没有，总有人需要这些服务。<br>
可装载内核模块，之前有提到一嘴，就是用来提供linux可拓展性的机制.<br>
LKMs的特点就是“即插即用”，你想使用的时候，就把它装载进内核空间，如果你不想用，就把它从内核空间里卸载，这样既不会占用资源，也能满足广大用户的需求，完美！<br>
>LKMs与用户态可执行文件一样都采用ELF格式，但是LKMs运行在内核空间，且无法脱离内核运行

有3个命令是专门用于LKMs的
>insmod xxx.ko 装载 （root）<br>
>lsmod 列出内核中以装载的ko文件<br>
>rmmod xxx.ko 卸载ko （root）<br>

## 5.内核内存管理<br>
kernel里有2个内存管理器，buddy system和slab allocator。
### buddy system<br>
linux的最底层内存管理器，直接管理物理内存，其实越底层的东西，设计思想也越简单<br>
buddy system 有几个简单的概念需要实现知道<br>
#### page<br>
buddy system管理内存的最小单位。
#### pageblock<br>
一串连续的物理页面被叫做pageblock（注意是连续）<br>
#### page order<br>
跟pageblock有关<br>
比如一个pageblock是由4个连续的page组成的，那么它的阶数就是2.<br>
如果一个pageblock是由1024个连续的page组成的，它的阶数就是10.<br>
#### 迁移类型<br>
page有主要分成 可迁移的和不可迁移的 两种。<br>
> 可迁移页面:指的是物理页面可以在不被用户感知的情况下，迁移到其他物理页面。比如用户空间使用的页面，可以修改虚实地址映射表，将虚拟地址偷偷映射到其他物理页面，用户感知不到物理页面的变化。
> 不可迁移页面：物理页面不可移动。比如分配个设备的页面，由于设置直接访问物理地址，所以页面不可移动。

其实有可迁移和不可迁移主要是为了减少内存碎片（外部碎片）。

#### buddy system分配过程<br>
在buddy ststem初始化后，一般情况下最高阶为10，物理内存划分如下:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220617145212.png)

***现在申请一个阶为0的pageblock*** <br>
分配页面要到最合适的链表上去分配，首选当然是从order为0的链表上分配。不巧的是，order为0的链表是空的，那么只能去order为1的链表上分配，更不巧的是order为1的链表也是空的，后面依次查询order为2-9的链表，都是空的。最终只能在order为10的链表上分配一组pages，并将这组pages从空闲链表删除，也即移除了buddy system。
然而，实际需要的是一个page，现在分配到的确是1024个pages，这要如何处理呢？其实很简单，剩余的1023个pages重新回到buddy system，重新回到buddy system遵循尽可能回到order较大的链表的原则。我们一步一步分析：<br>

>1023个pages回到order为10的链表是不可能的了，因为pages数量没有达到1024。
>1024个pages的后512个pages回到order为9的可移动迁移类型链表上，还剩余511个pages。
>511个pages的后256个pages回到order为8的可移动迁移类型链表上，还剩余255个pages。
>255个pages的后128个pages回到order为7的可移动迁移类型链表上，还剩余127个pages。
>最终order为9，8，7，6，5，4，3，2，1，0的可移动迁移类型链表上，都会增加一个成员。

分配完该page后，pages的分布图如下
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220617145559.png)

***申请一个阶为2的不可移动类型的pages*** <br>
问题来了，每个order的不可迁移类型的页面都是没有的，空空如也。<br>
buddy system 这时候提供了一个偷取机制。当分配不到某种迁移类型的物理页面时，会尝试从其他迁移类型的链表上偷取物理页面<br>
但是你偷也不能乱偷啊，如果你这时候只是偷了阶为2的那个pages，就会造成内存外部碎片的问题，因此偷取有如下原则:

>1. 从order最大的链表尝试偷。
>2. 偷的时候会将整个pageblock中所有的空闲物理页面都偷过去。pageblock的order一般对应最大的order，即10。
>3. 如果一个pageblock中有超过一半的物理页面被偷了，那么就会修改整个pageblock的迁移类型，当该pageblock的页面被释放时，会被添加到新的迁移类型对应的链表上去。这样一来，实际上相当于将整个pageblock都偷过去了。

那么当前场景下，会从order为10的可移动链表上偷一个成员，即偷取1024个物理页面。由于实际需要的是4个物理页面，1024个连续物理页面的后1020个物理页面会重新回到buddy system，但是此时会回到不可移动链表上。<br>
分配完成后，pages分配图如下：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220617150120.png)

***释放之前分配的order为0的page*** <br>
释放的page会尝试与其伙伴（即相邻的order为0的page）合并，由于其伙伴没有被分配，依然在buddy system中，所以二者可以合并为order为1的pages；order为1的pages会继续试图与其伙伴合并，当前上下文可以合并为order为2的pages；最终合并成order为1024个pages，回到了原点。<br>
page释放后，pages分布图如下：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220617150211.png)

### slab alloctor<br>
slab alloctor是buddy system的上一层内核分配管理器，它的出现主要是为了解决buddy system的一些其他问题，提高内存管理的效率。<br>
slab alloctor的目标如下：<br>
>1. 更小块的内存分配可以帮忙消除buddy allocator原本会造成的内部碎片问题
>>为了帮助消除buddy allocator造成的内部碎片问题，系统有维护两个cache集合，这些cache由细小的内存块组成，其大小从2^5 字节到2^17 字节不等
>>其中一个cache集合供DMA设备使用。这些cache叫做size-N和size-N(DMA)m其中N是指要分配的内存大小。可以调用kmallock()来分配这些cache中的memory。这样就解决了buddy allocator带来的low level page中的内部碎片问题。也就是如果只分配几个字节，如果没有slab allocator的话，也需要分配一个page

>2. 缓存常用的object因此系统不会在分配，初始化和销毁object上浪费时间。在Solaris上的Benchmarks显示使用slab allocator之后对于分配速度有很大的提升。
>>slab allocator的第二个任务是维护一些cache来分配常用的object。在内核中使用的许多结构体，初始化的时间甚至超过了分配时间。因此当一个新的slab被创建的时候，多个object被构造函数初始化后打包放入到slab中。如果一个object被释放了，它仍然以初始状态存放在slab中以以便object的分配能够加快。

>3. 通过将object地址与L1或者L2 cache对齐后可以更好的利用硬件缓存
>>slab allocator最后一个任务是利用硬件缓存。如果object打包放入slab后还有剩余空间，这些剩余空间将被用来将slab着色。slab着色是一种尝试让在不同slab中的object在硬件cache中使用不同的行的方案。在不同的slab中将object以不同的起始偏移来进行摆放，这就好像这些object在使用CPU硬件缓存的不同行一样来帮助保证从同一个slab分配出来的object不会相互从CPU硬件缓存中被刷掉。使用这种方案后，原本要浪费的空间被添加上了一种新的功能。下图显示了从buddy allocator中分配出来的一个page如何被用来存储与L1 CPU缓存对齐的object

#### slab allocator的版本<br>
slab有3个版本:

1. slab（另外两个机制的基础,最原始设计）
2. slob（嵌入式专用，极简设计）
3. slub（slab改进版）

#### slub数据结构<br>
更详细的内容可以看看
[https://zhuanlan.zhihu.com/p/490588193](https://zhuanlan.zhihu.com/p/490588193)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220617194314.png)
这张图很好！把slub的结构中的关系画了出来<br>
因为这个知乎写的很好，我就不哗众取宠了。<br>
这里主要写一下结构的概念:
>1. slab:slab其实就是从buddy system申请来的pageblock（当然会对这个pageblock做一些变量添加，方便控制资源,比如slab内部肯定是有关于空闲object的链表的），slab allocator用slab来称呼这个pages。
>2. node:有人可能会好奇buddy system中的node是拿来干嘛的，***node的数量是根据你电脑的内存控制器（memory controller）的数量来决定的*** 有几个memory controller就有几个node(内存控制器是一个物理的设备)
>3. slab allocator会将slab（也就是pageblock）作切分，化成若干个object（可以理解为glibc里的chunk）。
>4. kmem_cache:管理若干slab的结构体
>5. keme_cache_node结构体就是描述一个node里的slab使用情况的结构体（包括空闲的slab）
>6. kmem_cache管理的slab可以来自不同的node，因此kmem_cache里有kmem_cache_node的指针数组
>> kmem_cache_node结构体中有full链表（该链表上的slub没有objects可用），和partial链表（slub有部分或全部objects可用）<br>
>7. kmem_cache_cpu是一个可用的slab，是cpu私有变量，有了它在申请内存时就不需要加锁来影响分配效率。<br>

#### slub分配过程<br>
>1.从 kmem_cache_cpu里查看是否有空闲object，有就返回，没有进入第2步<br>
>2.将当前kmem_cache_cpu指定的slub加入到kmem_cache_node的full链表上，并尝试从 partial 链表上取一个 slub 挂载到 kmem_cache_cpu 上，然后再取出空闲对象返回<br>
>3.若 kmem_cache_node 的 partial 链表也空了，那就向 buddy system 请求分配新的内存页，划分为多个 object 之后再给到 kmem_cache_cpu，取空闲对象返回上层调用<br>

#### slub释放过程<br>
>1.若被释放 object 属于 kmem_cache_cpu 的 slub，直接使用头插法插入当前 CPU slub 的 freelist<br>
>2.若被释放 object 属于 kmem_cache_node 的 partial 链表上的 slub，直接使用头插法插入对应 slub 的 freelist<br>
>3.若被释放 object 属于 kmem_cache_node 的 full 链表上的 slub，则其会成为对应 slub 的 freelist 头节点，且该 slub 会从 full 链表迁移到 partial 链表<br>

### mmap、brk怎么申请内存<br>
glibc的一系列申请内存的操作离不开系统调用mmap、(s)brk<br>
[https://wsxk.github.io/linux%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E5%92%8C%E5%88%86%E9%85%8D/#malloc%E5%BA%95%E5%B1%82-brk%E5%92%8Cmmap](https://wsxk.github.io/linux%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80%E5%92%8C%E5%88%86%E9%85%8D/#malloc%E5%BA%95%E5%B1%82-brk%E5%92%8Cmmap)

mmap、brk的内存是从哪来的？<br>
>前文提到过，程序可以通过系统调用进入内核态。***在调用mmap、brk的系统调用后，进入内核态，内核会使用slab allocator分配内存，slab allocator会向buddy system申请内存。***

你一次性malloc申请超大内存（甚至超过物理内存大小)怎么办?<br>
>你当kernel设计师是傻子么，肯定有安全机制来制止这种行为呀。 ***这里关注的只是分配的流程*** 

## 6.kernel保护机制<br>
### 1.KASLR(kernel address space layout randomize,内核地址空间随机化)<br>
与用户态程序的ASLR相类似——在内核镜像映射到实际的地址空间时加上一个偏移值，但是内核内部的相对偏移其实还是不变的<br>
在未开启KASLR保护机制时，内核代码段的基址为 0xffffffff81000000 ，direct mapping area 的基址为 0xffff888000000000

### 2.FGKASLR<br>
KASLR 虽然在一定程度上能够缓解攻击，但是若是攻击者通过一些信息泄露漏洞获取到内核中的某个地址，仍能够直接得知内核加载地址偏移从而得知整个内核地址布局，因此有研究者基于 KASLR 实现了 FGKASLR，***以函数粒度重新排布内核代码*** (很高大上，但是感觉会大大降低系统的启动速度)

### 3.STACK PROTECTOR<br>
类似于用户态程序的 canary，通常又被称作是 stack cookie，用以检测是否发生内核堆栈溢出，若是发生内核堆栈溢出则会产生 kernel panic<br>
内核中的 canary 的值通常取自 gs 段寄存器某个固定偏移处的值

### 4.SMAP/SMEP（Supervisor Memory Access/Execution Prevention）<br>
管理模式访问保护和管理模式执行保护(这两种保护通常是同时开启的，用以阻止内核空间直接访问/执行用户空间的数据， ***完全地将内核空间与用户空间相分隔开***，用以防范ret2usr（return-to-user，将内核空间的指令指针重定向至用户空间上构造好的提权代码）攻击)

### 5.KPTI(kernel page-table isolation,内核页表隔离)<br>
内核空间和用户空间使用不同的页表集<br>
在这两张页表上都有着对用户内存空间的完整映射，但在用户页表中只映射了少量的内核代码（例如系统调用入口点、中断处理等），而只有在内核页表中才有着对内核内存空间的完整映射. **KPTI 同时还令内核页表中属于用户地址空间的部分不再拥有执行权限**

### 6.heap保护<br>
#### ①Hardened Usercopy<br>
hardened usercopy 是用以在用户空间与内核空间之间拷贝数据时进行越界检查的一种防护机制，主要检查拷贝过程中对内核空间中数据的读写是否会越界

#### ②Hardened freelist<br>
类似于 glibc 2.32 版本引入的保护，在开启这种保护之前，slub 中的 free object 的 next 指针直接存放着 next free object 的地址，攻击者可以通过读取 freelist 泄露出内核线性映射区的地址，在开启了该保护之后 free object 的 next 指针存放的是由以下三个值进行异或操作后的值
>当前 free object 的地址<br>
>下一个 free object 的地址<br>
>由 kmem_cache 指定的一个 random 值<br>

#### ③Random freelist<br>
这种保护主要发生在 slub allocator 向 buddy system 申请到页框之后的处理过程中，对于未开启这种保护的一张完整的 slub，其上的 object 的连接顺序是线性连续的，但在开启了这种保护之后其上的 object 之间的连接顺序是随机的，这让攻击者无法直接预测下一个分配的 object 的地址
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/kernel/20220617161458.png)
**需要注意的是这种保护发生在slub allocator 刚从 buddy system 拿到新 slub 的时候，运行时 freelist 的构成仍遵循 LIFO**

# reference<br>
[https://arttnba3.cn/2021/02/21/NOTE-0X02-LINUX-KERNEL-PWN-PART-I/#%E4%B8%AD%E6%96%AD](https://arttnba3.cn/2021/02/21/NOTE-0X02-LINUX-KERNEL-PWN-PART-I/#%E4%B8%AD%E6%96%AD)

[https://blog.csdn.net/digi2020/article/details/122534577](https://blog.csdn.net/digi2020/article/details/122534577)

[http://m.blog.chinaunix.net/uid-28772045-id-3672709.html](http://m.blog.chinaunix.net/uid-28772045-id-3672709.html)

[https://ja.wikipedia.org/wiki/Tty](https://ja.wikipedia.org/wiki/Tty)

[https://www.cnblogs.com/sparkdev/p/11460821.html](https://www.cnblogs.com/sparkdev/p/11460821.html)

[https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95](https://zh.wikipedia.org/wiki/%E4%B8%BB%E5%BC%95%E5%AF%BC%E8%AE%B0%E5%BD%95)

[https://zh.m.wikipedia.org/zh-hans/GNU_GRUB](https://zh.m.wikipedia.org/zh-hans/GNU_GRUB)

[https://www.zhihu.com/question/431255056](https://www.zhihu.com/question/431255056)

[https://zhuanlan.zhihu.com/p/360683396](https://zhuanlan.zhihu.com/p/360683396)

[https://blog.csdn.net/liuhangtiant/article/details/81043815](https://blog.csdn.net/liuhangtiant/article/details/81043815)

[https://blog.csdn.net/weixin_38537730/article/details/104520736](https://blog.csdn.net/weixin_38537730/article/details/104520736)

[https://zhuanlan.zhihu.com/p/370208909](https://zhuanlan.zhihu.com/p/370208909)

[https://blog.csdn.net/mbdong/article/details/121994275](https://blog.csdn.net/mbdong/article/details/121994275)

[https://blog.csdn.net/grabtalk520/article/details/81089557](https://blog.csdn.net/grabtalk520/article/details/81089557)

[https://zhuanlan.zhihu.com/p/478931356](https://zhuanlan.zhihu.com/p/478931356)

[https://blog.csdn.net/zwjyyy1203/article/details/97105647](https://blog.csdn.net/zwjyyy1203/article/details/97105647)

[https://zhuanlan.zhihu.com/p/490588193](https://zhuanlan.zhihu.com/p/490588193)

[https://blog.csdn.net/TABE_/article/details/122396297](https://blog.csdn.net/TABE_/article/details/122396297)