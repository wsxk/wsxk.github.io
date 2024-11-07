---
layout: post
tags: [kernel_pwn]
title: "linux内核基础 四 控制寄存器"
date: 2022-10-19
author: wsxk
comments: true
---

`更新于2022-10-19`<br>
PS:请学习完前三章后观看本章<br>

- [cr0](#cr0)
- [cr1](#cr1)
- [cr2](#cr2)
- [cr3](#cr3)
  - [KPTI(Kernel Page Table Isolation)](#kptikernel-page-table-isolation)
- [cr4](#cr4)
  - [SMAP/SMEP](#smapsmep)
- [references](#references)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>



linux内核定义的控制寄存器（control register）共有5个，分别为`cr0` `cr1`
`cr2` `cr3` `cr4`，这5个控制寄存器在内核中扮演了非常重要的角色，需要牢记<br>

## cr0<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/o_211023151018_4-5.png)<br>
cr0寄存器中的 `PE (page enable)` 表示模式，1表示保护模式，0表示实模式。<br>
cr0寄存器中的 `PG` 表示是否开启分页，`PG=1 PE=1`才开启分页模式，其他情况都不能开启<br>

## cr1<br>
cr1寄存器被保留了，暂时没有作用<br>

## cr2<br>
cr2寄存器用于存储触发缺页异常的线性地址（虚拟地址）
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/o_211023151024_4-6.png)<br>

## cr3<br>
cr3寄存器用于保存`页全局目录表PGD（page global directory)`的地址<br>
linux系统采用四级页表`PGD->PUD->PMD->PTE`<br>
### KPTI(Kernel Page Table Isolation)<br>
KPTI是内核用于隔离kernel和user的`PGD`的机制。<br>
显然，开启了该保护是由于修改了cr3寄存器的值来更换 不同的页表<br>
但是为了提高切换页表的速度，kernel页表和user页表是很接近的<br>
`user_pgd_addr = kernel_pgd_addr + 0x1000`<br>
反应到cr3寄存器上即翻转`cr3[12]`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/Rm8Ti9MpVUZ7fPK.png)<br>
这里需要知道的是 ***kernel页表保存了用户态和内核态空间的完整映射***。<br>
而  ***用户态页表只保留了用户态空间的完整映射和部分内核态空间的映射（系统调用入口点、中断处理等等）***  <br>
对于绕过kpti的保护，我们只需要执行内核中的`swapgs_restore_regs_and_return_to_usermode`函数来返回用户态即可（不使用swapgs、iretq）<br>
这个函数的功能相当于<br>
```
mov        rdi, cr3
or         rdi, 0x1000
mov     cr3, rdi
pop        rax
pop        rdi
swapgs
iretq
```
在rop绕过时需要执行如下操作<br>
```
↓   wapgs_restore_regs_and_return_to_usermode
    0 // padding
    0 // padding
    user_shell_addr
    user_cs
    user_rflags
    user_sp
    user_ss
```
感谢[arttnba3 blog](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#%E8%BF%94%E5%9B%9E%E7%94%A8%E6%88%B7%E6%80%81-with-KPTI-bypass)
详细描述了kpti及其绕过手法<br>

## cr4<br>
cr4寄存器结构如下<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/2520882-20220305153025181-197890540.png)<br>
### SMAP/SMEP<br>
其中需要我们关注的是`cr4[20]`和`cr4[21]`，标志位为1时表示开启SMAP/SMEP，标志位为0时表示关闭。<br>
`SMAP(Supervisor Mode Access Prevention)/SMEP(Supervisor Mode Execution Prevention)` SMAP开启时，kernel无法访问user地址的内存，SMEP开启时，kernel无法执行user地址的代码。<br>
但是显然，如果内核代码存在栈溢出问题可以让我们使用rop，或许我们可以利用rop修改cr4寄存器的 SMAP/SMEP标志位来使得kernel可以访问并执行 user的代码。<br>



## references<br>
[https://www.cnblogs.com/HcyRcer/p/16559321.html](https://www.cnblogs.com/HcyRcer/p/16559321.html)<br>
[arttnba3 blog](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#%E8%BF%94%E5%9B%9E%E7%94%A8%E6%88%B7%E6%80%81-with-KPTI-bypass)
详细描述了kpti及其绕过手法