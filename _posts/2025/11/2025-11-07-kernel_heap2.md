---
layout: post
tags: [kernel_pwn]
title: "kernel heap 2: 内核堆利用技巧综述"
author: wsxk
date: 2025-11-07
comments: true
---

- [1. kernel heap使用范式](#1-kernel-heap使用范式)
- [2. 内核堆漏洞](#2-内核堆漏洞)
  - [3.1 oob(out of boundry)：堆溢出](#31-oobout-of-boundry堆溢出)
  - [3.2 UAF](#32-uaf)
  - [3.3 overlapping allocation](#33-overlapping-allocation)
- [3.  泄露内核地址](#3--泄露内核地址)
  - [3.1 Oops泄露kernel地址绕过kaslr](#31-oops泄露kernel地址绕过kaslr)
- [4. 内核堆利用技巧](#4-内核堆利用技巧)
  - [4.1 Heap Spraying —— anit-freelist\_randomization](#41-heap-spraying--anit-freelist_randomization)
    - [4.1.1 OOB+Heap Spraying破解freelist\_randomization](#411-oobheap-spraying破解freelist_randomization)
  - [4.2 modprobe\_path —— 以内核权限执行任意文件](#42-modprobe_path--以内核权限执行任意文件)
  - [4.2.1 Oops泄露内核地址+UAF修改next\_ptr实现任意地址分配修改modprobe\_path](#421-oops泄露内核地址uaf修改next_ptr实现任意地址分配修改modprobe_path)
  - [4.3 堆布局构造](#43-堆布局构造)



# 1. kernel heap使用范式<br>
```c
cachep = (kmem_cache *)kmem_cache_create("kheap_obj", 472LL, 0LL, 84156416LL, 0LL);//大小为472
v6 = kmem_cache_alloc(cachep, 0x400CC0LL);//从这个kmem_cache中分配一个slot
kmem_cache_free(cachep, filp->private_data);//释放右侧chunk，归入cahchep中。
kmem_cache_destroy(cachep); //摧毁kmem_cache
```

# 2. 内核堆漏洞<br>
通常，内核堆漏洞分为如下三种：<br>
## 3.1 oob(out of boundry)：堆溢出<br>
顾名思义，其实就是在一个slot中填充多于其大小的内容，覆盖下一个slot中的其他值。<br>
只有这个漏洞通常能够泄露下一个slot中的信息（如果有机密信息的话）<br>

## 3.2 UAF<br>
uaf无需多说了。

## 3.3 overlapping allocation<br>
同理。<br>

# 3.  泄露内核地址<br>
## 3.1 Oops泄露kernel地址绕过kaslr<br>
触发Oops后的情景如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251023001013.png)
`R10的值ffffffff82a58c20为内核地址段，R12的值0xffff8880043a7000为物理映射区域，但是实际上，它会指向kernel的heap基址！`<br>


# 4. 内核堆利用技巧<br>
通常情况，找到了漏洞是以远远不够的。如何利用漏洞达成目的才是重点。为了达成目的，我们需要学习漏洞的利用手段.<br>

## 4.1 Heap Spraying —— anit-freelist_randomization<br>
heap spraying 是一个常见的内核堆利用技术，中文名堆喷射。**在kernel heap当中，堆的布局往往是不可知的，最主要的原因是开启了`freelist_randomization`选项（默认开启）**<br>
比如下图是一个slab中常见的object布局:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251028222255.png)
对于攻击者而言，无法确认下一个分配到的object位于哪个位置。（除非你的堆溢出能够溢出非常多，且覆盖正常object不会对kernel运行造成影响）。<br>
此时可以考虑利用`heap spraying`技术，我们可以申请多个object，让kernel耗尽当前kmem_cache_cpu中的slab，并让其重新申请一个slab：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251028223704.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251028223729.png)
此时再申请受害object后，堆溢出就容易许多了。<br>

应用场景:<br>
```
你有一个堆溢出读 / 写，但是堆布局对你而言是不可知的（比如说开启了 SLAB_FREELIST_RANDOM（默认开启）），你可以预先喷射大量特定结构体，从而保证对其中某个结构体的溢出。
```
### 4.1.1 OOB+Heap Spraying破解freelist_randomization<br>
前提：关闭kaslr，通过利用OOB漏洞，执行一次任意地址函数调用，且rdi寄存器是一个指针（不可改变），指向的内存区域可控。<br>
办法:申请多个slot，每个slot都覆盖相邻slot存放的函数地址，执行`commit_creds(rdi)`。rdi指向的内存区域，抄袭init_cred的内容<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251029222456.png)
事实证明，**拷贝init_cred的内容后，`commit_creds(rdi)`仍然成立，另外内核中所谓的原子变量，其实只指导如何操作这些变量，实际存储时和正常数据没有区别**<br>

## 4.2 modprobe_path —— 以内核权限执行任意文件<br>
modprobe是最初由Rusty Russell编写的Linux程序，用于在Linux内核中添加可加载的内核模块。实际上，当我们在Linux内核中安装或卸载新模块时，就会执行这个程序。它的路径是一个内核全局变量，默认为`/sbin/modprobe`，我们可以通过运行以下命令来检查它：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251104230614.png)
modprobe_path可以利用的原因如下:<br>
**1. 当我们执行具有未知文件类型的文件时，内核将执行存储在modprobe_path路径的程序。更准确地说，如果我们针对系统未知文件签名（魔术头）的文件调用execve()，则会产生以下调用，最终调用到modprobe：**<br>
```
（1）do_execve()
（2）do_execveat_common()
（3）bprm_execve()
（4）exec_binprm()
（5）search_binary_handler()
（6）request_module()
（7）call_modprobe()
```
**2. modprobe的路径（默认情况下为/sbin/modprobe）存储在内核本身的modprobe_path符号以及可写页面中。**<br>
我们可以通过读取/proc/kallsyms来获取其地址（由于KASLR机制，因此这个地址各不相同）：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251104230729.png)


***换句话说，如果我们有任意地址写的原语，那么可以将`modprobe_path`修改为我们想要执行的文件的路径，随后执行一下未知文件头的文件，即可完成以内核身份执行`modprobe_path`所指向的文件***<br>

## 4.2.1 Oops泄露内核地址+UAF修改next_ptr实现任意地址分配修改modprobe_path<br>
前提: 开启kaslr，存在UAF漏洞。kheap中存在函数地址，可执行一次该地址的调用，且rdi寄存器是一个指针（不可改变），指向的内存区域可控。<br>
办法：<br>
1. 首先利用UAF泄露`free_list ptr`得到内核堆地址；<br>
2. 然后利用UAF漏洞修改object中的`free_list ptr`，导致分配的内存能够操纵函数地址；<br>
3. 将函数地址改为0，造成kernel Oops泄露地址信息<br>
4. 重新利用UAF漏洞修改object中的`free_list ptr`，使其分配到`modprobe_path`附近，将`modprobe_path`改为我们想要的路径，比如`/tmp/exp`。<br>
5. 创建`/tmp/exp`文件，在其中写入我们想要执行的命令<br>

`modprobe_path`也是内核常用的利用手段。<br>
`modprobe_path`有使用细节需要注意：<br>
```c
void get_flag(void){
    puts("[*] Returned to userland, setting up for fake modprobe");
    
    system("echo '#!/bin/sh\ncp /dev/sda /tmp/flag\nchmod 777 /tmp/flag' > /tmp/x");//细节1：运行bash脚本前必须添加上 #!/bin/sh 否则触发了也不执行。
    system("chmod +x /tmp/x");

    system("echo -ne '\\xff\\xff\\xff\\xff' > /tmp/dummy");
    system("chmod +x /tmp/dummy");

    puts("[*] Run unknown file");
    system("/tmp/dummy");//细节2： 运行未知文件要写全局路径，不要用./

    puts("[*] Hopefully flag is readable");
    system("cat /tmp/flag");

    exit(0);
}
```

## 4.3 堆布局构造<br>
在kernel heap场景当中，堆布局是非常困难的。`kmalloc`函数会从 `通用的kmalloc_kmem_cache`中返回对象。然而：**通用cache可以保存许多大小相似的不同对象类型，这意味着：所有进程都会通过kmalloc分配通用slot（syscall也非常经常需要分配内存）**<br>
在内核的堆当中，并不是所有slot都是平等的，slot也分三六九等！<br>
我们需要的内核slot对象应该有如下的特性:<br>
```
1. object size可控
2. 避免copy_to_user 和 copy_from_user的使用（避免先前提到的Hardened Usercopy检测）
3. 保护ojbect ptr/function ptr（覆盖后就可以任意地址执行，十分方便）
```
这里提到两个非常有用的结构体:`msg_msg`和`pipe_buffer`<br>







<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>