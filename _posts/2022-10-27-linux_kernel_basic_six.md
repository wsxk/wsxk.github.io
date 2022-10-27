---
layout: post
tags: [kernel_pwn]
title: "linux内核基础 六 gs/fs寄存器 copy_from/to_user原理"
date: 2022-10-27
author: wsxk
comments: true
---

- [gs/fs寄存器<br>](#gsfs寄存器)
- [copy_from_user/copy_to_user底层原理<br>](#copy_from_usercopy_to_user底层原理)
- [references<br>](#references)

## gs/fs寄存器<br>
众所周知，在内核态和用户态之间进行转换时，需要调用`swapgs`命令来切换gs段寄存器。<br>
用户态还有个奇奇怪怪的fs段寄存器，这俩东西到底有啥用?<br>

**用户态使用fs寄存器引用线程的glibc TLS和线程在用户态的stack canary；用户态的glibc不使用gs寄存器；应用可以自行决定是否使用该寄存器（这里存在潜在的、充满想象力的优化空间）。**
<br>

**内核态使用gs寄存器引用percpu变量和进程在内核态的stack canary；内核态不使用fs寄存器。** <br>
简单来说 用户态使用fs寄存器来获得线程相关的结构体以及`stack canary`。内核态使用gs寄存器来获得`percpu变量和stack canary`。<br>
~~那有大聪明可能会问了，那为啥还要swapgs呢，gs直接设定成内核态使用不就行了呗~~<br>
其实还是不行，真设定成这样，这意味着用户态可以根据gs寄存器获得内核信息，这还挺要命的<br>

## copy_from_user/copy_to_user底层原理<br>
这个问题的来源在于我知道了`smap/smep`保护之后，如果开启了这俩保护，`copy_from/to_user`该如何和用户态进行交互呢？毕竟都禁止了内核态对用户态的访问和执行了.<br>
出于好奇翻阅了一下linux的源码(6.0.5) `/linux/uaccess.h` 和`/include/linux/instrumented.h`<br>
```c
copy_from_user(void *to, const void __user *from, unsigned long n)
{
	if (check_copy_size(to, n, false))
		n = _copy_from_user(to, from, n);
	return n;
}
_copy_from_user(void *to, const void __user *from, unsigned long n)
{
	unsigned long res = n;
	might_fault();
	if (!should_fail_usercopy() && likely(access_ok(from, n))) {
		instrument_copy_from_user(to, from, n);
		res = raw_copy_from_user(to, from, n);
	}
	if (unlikely(res))
		memset(to + (n - res), 0, res);
	return res;
}
instrument_copy_from_user(const void *to, const void __user *from, unsigned long n)
{
	kasan_check_write(to, n);
	kcsan_check_write(to, n);
}
```
我们已经很接近问题的答案了，问题应该是在`raw_copy_from_user`身上。<br>
在`/arch/alpha/include/asm/uaccess.h`找到了该函数<br>
```c
static inline unsigned long
raw_copy_from_user(void *to, const void __user *from, unsigned long len)
{
	return __copy_user(to, (__force const void *)from, len);
}
```
.....到这里还是没有找到。<br>
但是还是从一些blog里得到了相关的信息<br>
在调用底层函数时，copy发生前，通常会使用某些`特殊指令`关闭相应保护，在copy发生后再重置保护。<br>
比如`STAC(Set AC Flag)和CLAC(Clear AC Flag)` 就是x64中用来关闭和开启`SMAP`的汇编指令<br>
## references<br>
[fs/gs寄存器的作用](https://zhuanlan.zhihu.com/p/435518616)<br>
[https://blog.csdn.net/oqqYuJi12345678/article/details/103568087](https://blog.csdn.net/oqqYuJi12345678/article/details/103568087)<br>
[https://blog.csdn.net/faxiang1230/article/details/105962793](https://blog.csdn.net/faxiang1230/article/details/105962793)<br>
[https://bbs.pediy.com/thread-261744.htm](https://bbs.pediy.com/thread-261744.htm)
<br>
