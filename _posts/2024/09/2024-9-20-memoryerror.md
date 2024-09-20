---
layout: post
tags: [pwn]
title: "memory errors"
date: 2024-9-20
author: wsxk
comments: true
---

- [1. introduction](#1-introduction)
- [2. C high-level problems](#2-c-high-level-problems)
  - [2.1 Trusting the Developer](#21-trusting-the-developer)
  - [2.2 Mixing Control Information and Data](#22-mixing-control-information-and-data)
  - [2.3 Mixing Data and Metadata](#23-mixing-data-and-metadata)
  - [2.4 Initialization and Cleanup](#24-initialization-and-cleanup)
- [3. Memory errors: hazard](#3-memory-errors-hazard)
- [4. Memory errors: cause](#4-memory-errors-cause)
  - [4.1 Classic Buffer Overflow](#41-classic-buffer-overflow)
  - [4.2 Signedness Mixups](#42-signedness-mixups)
  - [4.3 Integer Overflows](#43-integer-overflows)
  - [4.4 Off-by-one Errors](#44-off-by-one-errors)
- [5. Memory errors protection: Stack Canaries](#5-memory-errors-protection-stack-canaries)
  - [5.1 bypass Stack Canaries](#51-bypass-stack-canaries)
- [6. Memory errors protection: ASLR](#6-memory-errors-protection-aslr)
  - [6.1 bypass ASLR](#61-bypass-aslr)
  - [6.2 Disabling ASLR for local testing](#62-disabling-aslr-for-local-testing)


## 1. introduction<br>
**内存破坏的起源思想：如果一个程序允许某人覆盖他们不应该覆盖的内存怎么办**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240914140009.png)
`Mainstream compiled languages`指的是编译语言(c,c++,etc)<br>
`VM-based languages`指的是解释语言(java,python,etc)<br>
编译语言带来的内存安全问题虽然严重，但是编译语言(c)运行的速度是最快的，所以到现在为止，`c/c++`仍然无处不在。<br>
目前，**想要保持速度，又能内存安全的尝试，就是Rust**（仍然努力中）<br>

## 2. C high-level problems<br>
### 2.1 Trusting the Developer<br>
c语言是非常信任开发者的。<br>
```c
int a[3] = { 1, 2, 3 };
a[10] = 0x41;
// no problem!
```
像python语言，这种写法不会通过。<br>
```python
>>> a = [ 1, 2, 3 ]
>>> print a[10] = 0x41;
IndexError: list index out of range
```

### 2.2 Mixing Control Information and Data<br>
很好理解，比如在栈结构中，函数的返回地址与用户数据是相邻的。<br>

### 2.3 Mixing Data and Metadata<br>
主要体现在字符串上。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240915091327.png)

### 2.4 Initialization and Cleanup<br>
c不会帮你自动初始化一个变量的值，当然，也不会帮你清理值。<br>
```c
int a; //a的值未知，取决于栈中的值，不会初始化

char * b = malloc(20);
free(b); // free后，内存中的值也不会自动清除
```

## 3. Memory errors: hazard<br>
在现有一个内存破坏漏洞的情况下，我们可以做到如下的事情:<br>
```
1. Memory that doesn't influence anything. (Boring)

2. Memory that is used in a value to influence mathematical operations, conditional jumps, etc (such as the win variable).

3. Memory that is used as a read pointer (or offset), allowing us to force the program to access arbitrary memory.

4. Memory that is used as a write pointer (or offset), allowing us to force the program to overwrite arbitrary memory.

5. Memory that is used as a code pointer (or offset), allowing us to redirect program execution!
```

## 4. Memory errors: cause<br>
### 4.1 Classic Buffer Overflow<br>
非常经典的溢出问题。c语言并不会隐式得跟踪buffer的大小，所以简单的`overwrite`是很常见的。

### 4.2 Signedness Mixups<br>
标准c语言库中使用`unsigned int`来表示`size`，例如`read, memcmp, strncpy中的最后一个参数`，但是我们通常使用的整数为`int`类型<br>
```c
int main() {
  int size;
  char buf[16];
  scanf("%i", &size);
  if (size > 16) exit(1);  //输入-1，跳过步骤
  read(0, buf, size); // 读取2^32-1个字节
}
```
为什么这会是一个问题呢，主要在于汇编层面对于有符号和无符号整形的条件跳转指令检验不同<br>
```
1. 0xffffffff == -1, 0xfffffffe == -2, etc

2. signedness mostly matters during conditional jumps

3. cmp eax, 16; jae too_big
    unsigned comparison
    eax = 0xffffffff will result in checking 0xffffffff > 16 and a jump

4. cmp eax, 16; jge too_big
    signed comparison
    eax = 0xffffffff will result in checking -1 > 16, and no jump
```
**注意，在用gdb/strace调试这些指令时，可能会出现和程序正常执行时截然不同的结果**<br>

### 4.3 Integer Overflows<br>
整形溢出问题通常发生在计算size的时候。
```c
int main() {
  unsigned int size;
  scanf("%i", &size);
  char *buf = alloca(size+1);//如果输入为2**31 -1， size+1 = 0
  int n = read(0, buf, size);
  buf[n] = '\0';
}
```

### 4.4 Off-by-one Errors<br>
`off-by-one`通常发生在如下场景:<br>
```c
	int a[3] = { 1, 2, 3 };
	for (int i = 0; i <= 3; i++) a[i] = 0;
```
`off-by-one`只允许一字节的溢出，取决于场景，可能会造成恐怖后果。<br>


## 5. Memory errors protection: Stack Canaries<br>
为了对抗缓冲区溢出到程序的返回地址，研究人员们引入了`stack canaries`<br>
`stack canaries`主要做的是两件事:<br>
```
1. In function prologue, write random value at the end of the stack frame.
函数开头，在栈帧的末尾填入随机值

2. In function epilogue, make sure this value is still intact.
函数结尾，校验这个值是否是完整的
```

### 5.1 bypass Stack Canaries<br>
`stack canaries`真的是一个非常有效的防护手段，但是仍然有一些情境下可以绕过这个防护<br>

```
1. Leak the canary (using another vulnerability).
使用其他漏洞来泄露canary

2. Brute-force the canary (for forking processes).
对于类似
int main() {
    char buf[16];
    while (1) {
        if (fork()) { wait(0); }
        else { read(0, buf, 128); return; }
    }
}
的代码，能够爆破canary的值（8字节也只需要爆破256*8次）

3. jumping the canary (if the situation allows).
跳过canary完成返回地址的覆写，主要针对以下场景
int main() {
    char buf[16];
    int i;
    for (i = 0; i < 128; i++) read(0, buf+i, 1);
}
取决于程序的布局，你可以通过修改i值来绕过canary的写入，直接写入返回地址
```

## 6. Memory errors protection: ASLR<br>
`ASLR(Address Space Layout Randomrization,地址空间布局随机化)`也是内存破坏的常见防御手段。<br>
其核心思想在于**黑客们通常聚焦于把破坏指针，使其指向其他位置，那么要指向其他位置，就需要知道其他位置的内存地址**<br>
如果我们随机化程序的地址空间排布，要想攻击，顶多制造程序崩溃，想要控制程序的执行权限就变得困难了。<br>

### 6.1 bypass ASLR<br>
有一些场景下可以绕过`ASLR`：<br>
```
Method 1: Leak
The addresses still (mostly) have to be in memory so that the program can find its own assets.
Let's leak them out!

Method 2: YOLO
Program assets are page-aligned.
Let's overwrite just the page offset!
Requires some brute-forcing.
核心原理是系统中pages都是0x1000对齐的，程序段的空间一般都在一个page范围内，我们可以操纵前2个字节的偏移，来跳转到程序的其他位置。  这种情况下爆破只需爆破16位，还是很好爆的

Method 3 (situational): brute-force
int main() {
    char buf[16];
    while (1) {
        if (fork()) { wait(0); }
        else { read(0, buf, 128); return; }
    }
}
就比较难爆破，基本不现实
```

### 6.2 Disabling ASLR for local testing<br>
调试的时候我们不希望有`ASLR`,有办法可以禁用它。<br>
```
In pwntools:
pwn.process("./vulnerable_proram", aslr=False)

gdb will disable ASLR by default if has permissions to do so. NOTE: for SUID binaries, remove the SUID bit before using gdb (chmod or cp).

You can spin up a shell whose (non-setuid) children will all have ASLR disabled:
# setarch x86_64 -R /bin/bash
```
