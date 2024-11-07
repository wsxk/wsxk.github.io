---
layout: post
tags: [kernel_pwn]
title: "linux内核基础 二 物理内存映射区"
date: 2022-10-12
author: wsxk
comments: true
---

`更新于2022-10-13`<br>
PS: 好学者请先学习完[linux内核基础 一](https://wsxk.github.io/linux_kernel_basic_one/)<br>
并完成对应[习题](https://wsxk.github.io/qwb2018_core/)后学习该篇内容<br>

- [direct mapping of all physical memory](#direct-mapping-of-all-physical-memory)
- [linux内核内存分配函数](#linux内核内存分配函数)
	- [kmalloc](#kmalloc)
	- [kzalloc](#kzalloc)
	- [vmalloc](#vmalloc)
	- [\_\_get\_free\_pages:](#__get_free_pages)
- [task\_struct](#task_struct)
- [进程内核栈](#进程内核栈)
	- [pt\_regs](#pt_regs)
	- [通过task\_struct寻找内核栈（32位）](#通过task_struct寻找内核栈32位)
	- [通过内核栈找task\_struct（32位）](#通过内核栈找task_struct32位)
	- [64位下cpu的task\_struct/内核栈索引](#64位下cpu的task_struct内核栈索引)
- [内核态和用户态转变（again）](#内核态和用户态转变again)
	- [用户态-\>内核态](#用户态-内核态)
	- [内核态-\>用户态](#内核态-用户态)
- [references](#references)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## direct mapping of all physical memory<br>
[https://elixir.bootlin.com/linux/latest/source/Documentation/x86/x86_64/mm.rst](https://elixir.bootlin.com/linux/latest/source/Documentation/x86/x86_64/mm.rst)里提出了官方的64位linux下虚拟内存布局<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013163946.png)<br>
本节我们重点关注的是其中一项<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013164029.png)<br>
`direct mapping of all physical memory(page_offset_base)`这一项，其表明的意思是，这段虚拟内存区域直接映射了整个`物理内存`，换句话说，这一段区域的地址和物理内存地址存在线性关系（`virtual_addr = physical_addr + 0xffff888000000000`<br>
从这里也可以看出linux64位支持的最大的物理内存为`64TB`<br>
同时，`direct mapping of all physical memory(page_offset_base)`位于内核虚拟地址空间中，那么，对于一个用户态的虚拟地址--其对应的物理地址，在内核态中也有一个虚拟地址对应。`即同一个物理内存区域，可以同时通过用户态虚拟地址和内核态虚拟地址进行读写`<br>
这里存在一个攻击手段，即在用户态的虚拟地址写入shellcode/rop_chain，通过泄露内核态地址，可以实现在内核态中执行，非常的牛逼<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013165714.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013165732.png)

## linux内核内存分配函数<br>
### kmalloc<br>
kmalloc申请的虚拟内存地址位于`direct mapping of all physical memory`物理内存映射区域，在物理上连续，于真实物理地址的差值为一个定值，存在简单的转换关系，申请大小不能超过128kb<br>
### kzalloc<br>
kzalloc = kmalloc + 清空内存（为0）<br>
### vmalloc<br>
vmalloc() 函数则会在虚拟内存空间给出一块连续的内存区，但这片连续的虚拟内存在物理内存中并不一定连续。由于 vmalloc() 没有保证申请到的是连续的物理内存，因此对申请的内存大小没有限制，如果需要申请较大的内存空间就需要用此函数了。<br>
### __get_free_pages:<br>
于kmalloc一样，申请的虚拟地址位于`direct mapping of all physical memory`区域，是提供给调用者最底层的内存分配函数，基于`buddy system`实现，同样是连续的物理内存。分配粒度为页<br>

## task_struct<br>
对于OS来说，为了能够方便的管控进程的运行情况，使用一个叫做`PCB(Process Control Block)`的数据结构来记录进程的`所有`信息<br>
当启动一个进程后，OS会创建一个PCB结构体（声明为`task_struct`），当进程结束后，才会撤销。<br>
task_struct在Linux中的`<include/linux/sched.h>`中被定义。<br>
可以在该网址中越读源码:[https://elixir.bootlin.com/linux/v6.0-rc5/source/include/linux/sched.h](https://elixir.bootlin.com/linux/v6.0-rc5/source/include/linux/sched.h)<br>
task_struct要记录进程的所有信息，其结构也极其复杂，对于本次学习，我们只需要关注其中2个定义
```c
struct task_struct {
    struct thread_info	thread_info;
    ...
    void* stack;
    ...
};
```
thread_info 结构体中，我们需要知道其中一个定义
```c
struct thread_info{
    ...
    struct task_struct * task;
};
```
这几个定义具体有什么用，接下来会讲<br>

## 进程内核栈<br>
众所周知，栈(stack)是当今计算机系统中不可或缺的结构之一，其实，一个进程的栈分为`用户栈`和`内核栈`。在运行用户态代码时，使用的是用户栈，当进程通过`系统调用`等操作陷入内核态时，在内核态运行代码使用的是内核栈。<br>
内核栈的定义如下:
```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```
其中`THREAD_SIZE`的定义如下（`arch/x86/include/asm/page_64_types.h` `arch/x86/include/asm/page_32_types.h`）:
```c
//ARM架构 , 8K
#define THREAD_SIZE_ORDER	1
#define THREAD_SIZE		(PAGE_SIZE << THREAD_SIZE_ORDER)
#define THREAD_START_SP		(THREAD_SIZE - 8)

//ARM64架构, 16K
#define THREAD_SIZE		16384
#define THREAD_START_SP		(THREAD_SIZE - 16)

//X86_64, 16K
#define THREAD_SIZE_ORDER	(2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

### pt_regs<br>
pt_regs是一个寄存器组结构，用于在用户态和内核态进行转换时，保存上下文（context）所用<br>
```c
struct pt_regs { //x86 32bits
	unsigned long bx;
	unsigned long cx;
	unsigned long dx;
	unsigned long si;
	unsigned long di;
	unsigned long bp;
	unsigned long ax;
	unsigned short ds;
	unsigned short __dsh;
	unsigned short es;
	unsigned short __esh;
	unsigned short fs;
	unsigned short __fsh;
	/*
	 * On interrupt, gs and __gsh store the vector number.  They never
	 * store gs any more.
	 */
	unsigned short gs;
	unsigned short __gsh;
	/* On interrupt, this is the error code. */
	unsigned long orig_ax;
	unsigned long ip;
	unsigned short cs;
	unsigned short __csh;
	unsigned long flags;
	unsigned long sp;
	unsigned short ss;
	unsigned short __ssh;
};
```
64位定义如下:
```c
struct pt_regs { //x86-64 64bits
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
	unsigned long r15;
	unsigned long r14;
	unsigned long r13;
	unsigned long r12;
	unsigned long bp;
	unsigned long bx;
/* These regs are callee-clobbered. Always saved on kernel entry. */
	unsigned long r11;
	unsigned long r10;
	unsigned long r9;
	unsigned long r8;
	unsigned long ax;
	unsigned long cx;
	unsigned long dx;
	unsigned long si;
	unsigned long di;
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
	unsigned long orig_ax;
/* Return frame for iretq */
	unsigned long ip;
	unsigned long cs;
	unsigned long flags;
	unsigned long sp;
	unsigned long ss;
/* top of stack page */
};
```

既然面临着 用户栈和内核栈的转换，那么OS就需要知道用户栈和内核栈的“位置”，即如何查询到用户栈和内核栈<br>
### 通过task_struct寻找内核栈（32位）<br>
之前提到了task_struct中有 `void * stack`值，通过如下代码找到内核栈<br>
```c
static inline void *task_stack_page(const struct task_struct *task)
{
	return task->stack;
}
```
可以通过如下的代码索引到pt_regs的位置<br>
```c
//processor.h	(arch\x86\include\asm)
#define task_pt_regs(task) \
({									\
	unsigned long __ptr = (unsigned long)task_stack_page(task);	\
	__ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;		\
	((struct pt_regs *)__ptr) - 1;					\
})
```
从上述代码也可以看到，pt_regs结构体被放置在`内核栈的下方`
因此可以通过task_struct方便地找到内核栈和pt_regs的位置,如下图所示<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013201914.png)
### 通过内核栈找task_struct（32位）<br>
如上图所示，`内核栈的上方`（栈是自底向上增长的），存放着thread_info结构体，thread_info中存有指向task_struct的指针。<br>
### 64位下cpu的task_struct/内核栈索引<br>
64位的cpu中有一个`Per-CPU`变量，用来存放task_struct的指针，因此不再需要thread_info进行索引<br>
32位和64位的不同由下图表示:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221013201800.png)<br>
图大多出自[内核栈和用户栈](https://blog.csdn.net/u012489236/article/details/116614606?ops_request_misc=&request_id=&biz_id=102&utm_term=linux%E5%86%85%E6%A0%B8%E6%A0%88%E5%B8%83%E5%B1%80&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-5-116614606.142%5Ev31%5Econtrol,185%5Ev2%5Econtrol&spm=1018.2226.3001.4187)<br>

## 内核态和用户态转变（again）<br>
现在有一定基础后，再来看看linux用户态和内核态的转变吧！<br>
### 用户态->内核态<br>
方式:
> 1. 系统调用
> 2. 异常（fault/trap）
> 3. 外设中断

在发生如上情况时，os主要做了以下几件事<br>
```
1. 切换gs寄存器 swapgs
2. 保存用户态栈帧信息（用户栈顶放入CPU独占变量，CPU独占变量里的内核栈顶放入rsp/esp寄存器中）
3. 保存用户态寄存器信息(push各个寄存器的值到内核栈上(pt_regs))
4. 通过汇编指令判断为32位/64位
5. 控制器转交内核，执行系统调用(sys_call_table)
```
### 内核态->用户态<br>
```
1. swapgs
2. iretq/sysretq
	user_shell_addr
	user_cs
	user_eflags
	user_sp
	user_ss
```

## references<br>
[arttnba3的blog](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#0x02-Kernel-ROP-ret2usr)<br>
[内核栈和用户栈](https://blog.csdn.net/u012489236/article/details/116614606?ops_request_misc=&request_id=&biz_id=102&utm_term=linux%E5%86%85%E6%A0%B8%E6%A0%88%E5%B8%83%E5%B1%80&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-5-116614606.142%5Ev31%5Econtrol,185%5Ev2%5Econtrol&spm=1018.2226.3001.4187)<br>
[kmalloc、kzalloc、vmalloc、__get_free_pages的区别](https://blog.csdn.net/alimingh/article/details/111942297)<br>
[linux内存布局（官方）](https://elixir.bootlin.com/linux/latest/source/Documentation/x86/x86_64/mm.rst)<br>
[一篇文章读懂mmap](https://zhuanlan.zhihu.com/p/366964820)<br>
