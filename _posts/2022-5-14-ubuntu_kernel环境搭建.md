---
layout: post
title: "ubuntu20 kernel环境搭建"
date:   2022-5-14
tags: [software_build]
comments: true
author: wsxk
---

最近准备入门linux kernel pwn。

kernel，pwn pwn人的最终归宿（我随便说的）。

因为pwn 就是对 计算机底层机制进行利用的技术

从 stack->heap->kernel，对pwn的深入学习，就是对计算机底层的深入学习。

所以为了学习 kernel pwn

咱们先从最基础的 kernel 环境搭建开始吧

可以直接看这位大佬写的blog。这位大佬写的非常好。

[https://arttnba3.cn/2021/02/21/NOTE-0X02-LINUX-KERNEL-PWN-PART-I](https://arttnba3.cn/2021/02/21/NOTE-0X02-LINUX-KERNEL-PWN-PART-I/#%E4%BA%8C%E3%80%81%E8%8E%B7%E5%8F%96busybox)

我写这个blog主要是把搭建过程提取出来，方便我日后回溯。

- [基本环境](#基本环境)
- [编译内核](#编译内核)
  - [1.下载内核源码](#1下载内核源码)
  - [2.解包并设置选项](#2解包并设置选项)
    - [可能遇到的问题](#可能遇到的问题)
- [安装busybox](#安装busybox)
- [构建磁盘镜像](#构建磁盘镜像)
  - [1. 构造文件系统的大致结构](#1-构造文件系统的大致结构)
  - [2.配置初始化脚本（方法一）](#2配置初始化脚本方法一)
  - [2.配置初始化脚本（方法二）](#2配置初始化脚本方法二)
  - [3.配置用户](#3配置用户)
- [打包文件系统](#打包文件系统)
  - [如果出现要在文件系统里新增内容怎么办？](#如果出现要在文件系统里新增内容怎么办)
- [qemu调试](#qemu调试)
  - [1. 配置启动脚本](#1-配置启动脚本)
- [调试内核](#调试内核)
  - [1.得到符号表](#1得到符号表)
  - [2.远程连接](#2远程连接)
- [参考](#参考)

## 基本环境

ubuntu 20.04 LTS

基本环境可以参考我的另一个博客

[ubuntu20虚拟机搭建](https://wsxk.github.io/ubuntu_%E5%AE%89%E8%A3%85%E5%B8%B8%E8%A7%84%E6%93%8D%E4%BD%9C/)

初此之外 你还需要安装其他的一些东西

    apt install git flex bison qemu-system 

## 编译内核

### 1.下载内核源码

[下载网址](https://www.kernel.org/)

[历史下载](https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/)

使用如下命令下载

    wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.11.tar.xz

（如果没有wget 可以直接 apt install wget）

### 2.解包并设置选项

    tar -xvf linux-5.11.tar.xz
    cd linux-5.11
    make menuconfig

注意 如果你不能进入 make menuconfig

可以看一下具体报错是什么（一般都是缺少什么东西，apt install xxx 补上就行，如果apt install xxx 也不行，把报错复制下来去百度也是能解决的）

成功后，会看到如下界面

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-14-ubuntu_kernel%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/1.png)

进入选项后，其实不需要你做什么，exit退出后保存配置到.config即可。

接下来

    make bzImage

编译成功后，在 

    linux-5.11/arch/x86/boot/

下会有bzImage 



#### 可能遇到的问题

如果你在编译时遇到下面的问题

    make[1]: *** No rule to make target 'debian/canonical-certs.pem', needed by 'certs/x509_certificate_list'.  Stop

使用如下命令

    gedit .config

然后用搜索功能搜索关键词 canonical-certs.pem，能找到这个选项后，把值改成“”，如下图。

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-14-ubuntu_kernel%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/2.png)

继续编译。

如果编译提示你缺少什么东西

一般是 apt install 就能解决

## 安装busybox

只编译了一个内核是不够的，我们还需要一个文件系统（busybox可以帮我们解决这个问题


    wget https://busybox.net/downloads/busybox-1.33.0.tar.bz2

    tar -jxvf busybox-1.33.0.tar.bz2

    make menuconfig

记得Settings —> Build static binary file (no shared lib)需要标上！！！！

！！！！！！！！！！！！！！！！！！！！！！

    make install

编译完成后，会生成_install 目录


##  构建磁盘镜像

### 1. 构造文件系统的大致结构

文件系统的大致结构其实也好懂。你的虚拟机使用 ls / 展现出来的就是文件系统的一般结构了。

    cd _install
    mkdir -pv {bin,sbin,etc,proc,sys,home,lib64,lib/x86_64-linux-gnu,usr/{bin,sbin}}
    touch etc/inittab
    mkdir etc/init.d
    touch etc/init.d/rcS
    chmod +x ./etc/init.d/rcS


### 2.配置初始化脚本（方法一）

    gedit etc/inttab

写入如下信息

    ::sysinit:/etc/init.d/rcS  # 指定初始化脚本
    ::askfirst:-/bin/ash
    ::ctrlaltdel:/sbin/reboot
    ::shutdown:/sbin/swapoff -a
    ::shutdown:/bin/umount -a -r
    ::restart:/sbin/init

接下来要写入 etc/init.d/rcS信息

    gedit etc/init.d/rcS

信息如下

    #!/bin/sh
    mount -t proc none /proc
    mount -t sys none /sys
    /bin/mount -n -t sysfs none /sys
    /bin/mount -t ramfs none /dev
    /sbin/mdev -s


### 2.配置初始化脚本（方法二）

    touch init
    gedit init

在init中写入如下信息

    #!/bin/sh

    mount -t proc none /proc
    mount -t sysfs none /sys
    mount -t devtmpfs devtmpfs /dev

    exec 0</dev/console
    exec 1>/dev/console
    exec 2>/dev/console

    echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
    setsid cttyhack setuidgid 1000 sh

    umount /proc
    umount /sys
    poweroff -d 0  -f

写入完成后

    chmod +x init

### 3.配置用户

    $ echo "root:x:0:0:root:/root:/bin/sh" > etc/passwd  # user root 
    $ echo "ctf:x:1000:1000:ctf:/home/ctf:/bin/sh" >> etc/passwd # user ctf
    $ echo "root:x:0:" > etc/group    # group root
    $ echo "ctf:x:1000:" >> etc/group # group ctf
    $ echo "none /dev/pts devpts gid=5,mode=620 0 0" > etc/fstab

## 打包文件系统

    find . | cpio -o --format=newc > ../../rootfs.cpio

### 如果出现要在文件系统里新增内容怎么办？

    首先需要解压镜像文件

    cpio -idv < ./rootfs.cpio #把内容解压到当前目录下

把内容添加进去后

    find . | cpio -o --format=newc > ../new_rootfs.cpio

重新打包即可

## qemu调试

### 1. 配置启动脚本

首先将先前的bzImage和rootfs.cpio放到同一个目录下，然后创建一个boot.sh文件

    touch boot.sh

写入内容

    #!/bin/sh
    qemu-system-x86_64 \
        -m 128M \
        -kernel ./bzImage \
        -initrd  ./rootfs.cpio \
        -monitor /dev/null \
        -append "root=/dev/ram rdinit=/sbin/init console=ttyS0 oops=panic panic=1 loglevel=3 quiet nokaslr" \
        -cpu kvm64,+smep \
        -smp cores=2,threads=1 \
        -netdev user,id=t0, -device e1000,netdev=t0,id=nic0 \
        -nographic \
        -s


一些解释

    -m：虚拟机内存大小

    -kernel：内存镜像路径

    -initrd：磁盘镜像路径

    -append：附加参数选项

        nokalsr：关闭内核地址随机化，方便我们进行调试

        rdinit：指定初始启动进程，/sbin/init进程会默认以/etc/init.d/rcS作为启动脚本
        loglevel=3 & quiet：不输出log

        console=ttyS0：指定终端为/dev/ttyS0，这样一启动就能进入终端界面

    -monitor：将监视器重定向到主机设备/dev/null，这里重定向至null主要是防止CTF中被人给偷了qemu拿flag

    -cpu：设置CPU安全选项，在这里开启了smep保护
    -s：相当于-gdb tcp::1234的简写（也可以直接这么写），后续我们可以通过gdb连接本地端口进行调试

随后运行 

    ./boot.sh

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-14-ubuntu_kernel%E7%8E%AF%E5%A2%83%E6%90%AD%E5%BB%BA/3.png)


## 调试内核

之前的boot.sh已经开通了远程端口 1234

接下来要怎么做？

### 1.得到符号表

[https://github.com/torvalds/linux/blob/master/scripts/extract-vmlinux](https://github.com/torvalds/linux/blob/master/scripts/extract-vmlinux)

下载这个文件（可以从文件系统里提取出 vmlinux，这个文件里有符号）

    ./extract-vmlinux ./bzImage > vmlinux

运行gdb时

    gdb vmlinux

这个文件里还要ROPgadget给你找

    ROPgadget --binary ./vmlinux > gadget.txt

### 2.远程连接

在gdb界面里

    set architecture i386:x86-64

    target remote localhost:1234

即可。



## 参考

[https://arttnba3.cn/2021/02/21/NOTE-0X02-LINUX-KERNEL-PWN-PART-I](https://arttnba3.cn/2021/02/21/NOTE-0X02-LINUX-KERNEL-PWN-PART-I/#%E4%BA%8C%E3%80%81%E8%8E%B7%E5%8F%96busybox)

[https://blog.csdn.net/OnlyLove_/article/details/122646605](https://blog.csdn.net/OnlyLove_/article/details/122646605)