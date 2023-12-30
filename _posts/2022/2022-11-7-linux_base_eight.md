---
layout: post
tags: [kernel_pwn]
title: "linux 内核基础 八 modprobe提权"
date: 2022-11-7
author: wsxk
comments: true
---

- [什么是modprobe<br>](#什么是modprobe)
- [modprobe和 insmod/rmmod的区别<br>](#modprobe和-insmodrmmod的区别)
- [如何利用modprobe进行内核提权<br>](#如何利用modprobe进行内核提权)
- [modprobe提权的利用条件<br>](#modprobe提权的利用条件)
- [references<br>](#references)

## 什么是modprobe<br>
modprobe是一个内置的linux系统命令，用于安装/卸载 LKM（loadable kernel module），它是一个内核全局变量，可以通过以下的命令进行查看：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221107192516.png)

## modprobe和 insmod/rmmod的区别<br>
众所周知，linux中还内置了另一组用于安装/卸载 LKM的命令 `insmod/rmmod`，modprobe和这组命令相比起来是有区别的。
> 1. modprobe可以解决 加载模块 的依赖关系，比如加载module a前必须加载module b，modprobe可以通过 /lib/modules/`uname -r`/modules.dep 查找依赖关系
> 2. modprobe 需要在/lib/modules/`uname -r`/ 下查找module，而insmod可以在当前目录下直接装载

## 如何利用modprobe进行内核提权<br>
modprobe之所以可以进行内核提权，一个很重要的原因是， **modprobe的路径（默认情况下是/sbin/modprobe）存储在内核本身的modprobe_path符号中，并且该页面是可写的**<br>
另一个原因是<br>
**在我们执行（exec）一个未知类型的文件时（magic number，魔数）（文件抬头没有被识别），系统会调用，会产生以下调用，最终调用到modprobe**<br>
```c
entry_SYSCALL_64()
    sys_execve()
        do_execve()
            do_execveat_common()
                bprm_execve()
                    exec_binprm()
                        search_binary_handler()
                            __request_module() // wrapped as request_module
                                call_modprobe()
```
其中 call_modprobe的源码如下（5.14）`kernel/kmod.c`
```c
static int call_modprobe(char *module_name, int wait)
{
    //...

    argv[0] = modprobe_path;
    argv[1] = "-q";
    argv[2] = "--";
    argv[3] = module_name;    /* check free_modprobe_argv() */
    argv[4] = NULL;

    info = call_usermodehelper_setup(modprobe_path, argv, envp, GFP_KERNEL,
                     NULL, free_modprobe_argv, NULL);

    if (!info)
        goto free_module_name;

    return call_usermodehelper_exec(info, wait | UMH_KILLABLE);
    //...
```
内核会以 ***root*** 权限来执行modprobe，如果我们能够劫持modprobe_path并将其指向我们自己的程序，那么就能成功完成利用<br>

## modprobe提权的利用条件<br>
> （1）知道modprobe_path的地址；<br>
> （2）知道kpti_trampoline(即swapgs_restore_regs_and_return_to_usermode函数)的地址，以便在覆写modprobe_path之后干净地返回到用户空间；<br>
> （3）有任意写入原语。<br>

## references<br>
[https://blog.csdn.net/vevenlcf/article/details/78884672](https://blog.csdn.net/vevenlcf/article/details/78884672)<br>
[https://blog.csdn.net/weixin_45619852/article/details/121102283](https://blog.csdn.net/weixin_45619852/article/details/121102283)<br>
[https://www.anquanke.com/post/id/232545](https://www.anquanke.com/post/id/232545)<br>