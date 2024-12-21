---
layout: post
tags: [pwn]
title: "race conditions"
author: wsxk
date: 2024-12-11
comments: true
---

- [1. 什么是race condition](#1-什么是race-condition)
- [2. races in filesystem](#2-races-in-filesystem)
  - [2.1 races in filesystem 原理](#21-races-in-filesystem-原理)
  - [2.2 提高rece condition成功概率的方法](#22-提高rece-condition成功概率的方法)
    - [2.2.1 方法一: nice](#221-方法一-nice)
    - [2.2.2 方法二: Path Complexity](#222-方法二-path-complexity)
  - [2.3 mitigation](#23-mitigation)
- [3. processes and threades](#3-processes-and-threades)
  - [3.1 进程和线程介绍](#31-进程和线程介绍)
  - [3.2 创建thread](#32-创建thread)
  - [3.3 libc和syscall接口的差异](#33-libc和syscall接口的差异)
- [4. Races in memory](#4-races-in-memory)
  - [4.1 Double Fetch](#41-double-fetch)
  - [4.2 General data races](#42-general-data-races)
- [5. Signals and reentrancy](#5-signals-and-reentrancy)


# 1. 什么是race condition<br>
在古早时期，CPU是单核的，但是你想在这颗CPU上运行多个进程，这意味着:<br>
```
用户感受到两个进程同时运行，但是实际上在某个时刻内，只有一个进程能被执行
只不过用户(人)感受不到这个变化
```
现代，CPU是多核的，但是:<br>
```
1. 进程数量还是多于核数
2. 内核因此还是需要决定什么时候调度哪些进程
3. 存储控制器的通道有限（四通道），存储媒介通道有限，网络通信是单通道的
```
这些限制都导致计算机无法同时运行所有进程。<br>
`race condition`出现的核心要旨是**计算设备的瓶颈导致并发的事件至少有一部分需要被序列化**<br>
**通常，没有隐式依赖或者程序显式的努力，执行顺序只能在进程内(一个线程)得到保证**<br>
这也导致一个问题，如果进程A和B都检查文件C的内容(check),符合条件就更新文件C(change)；然后进程A和B按照如下顺序被执行:<br>
```
1. A check #通过
2. B check #通过
3. A change # A改变了C的内容
4. B change # 讲道理此时不应该执行B，但是B还是能被执行，导致C的内容被改变了2次
```
这就是`race condition`!<br>
**要想利用race condition，我们需要在应用执行的薄弱时期，精细地影响其状态才行**<br>

# 2. races in filesystem<br>
书接上回，攻击者通过改变程序运行的状态，而程序假设它的状态没有改变,导致`race condition`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241212221549.png)
为了利用`race condition`,攻击者需要能够影响所说的环境，**Races in filesystem**就是一个很常见的例子。<br>
## 2.1 races in filesystem 原理<br>
文件系统是攻击者能够经常影响的程序环境的一部分。<br>
利用文件系统实施攻击，本质上是在进程运行时，操控进程运行所需的文件达到利用`race condition`的目的。<br>
考虑这段代码:<br>
```c
int main(int argc, char **argv) {
    int fd = open(argv[1], O_WRONLY | O_CREAT | O_TRUNC, 0755);//O_TRUNC 标志在打开文件时会丢弃文件原本的内容，即清0
    write(fd, "#!/bin/sh\necho SAFE\n", 20);
    close(fd);
    execl("/bin/sh", "/bin/sh", argv[1], NULL);
}
```
在open打开文件和实际执行/binsh之间存在空窗期可以利用！<br>
```
具体利用步骤:
1. gcc fs1.c -o fs1
2. 启动一个terminal， 运行 while /bin/true; do cp -v catflag asdf; done
3. 再启动一个terminal，运行 for i in $(seq 1 2000); do ./fs1 asdf; done | tee output
   运行 sort output | uniq -c 查看运行次数  
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241213210133.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241213211751.png)

## 2.2 提高rece condition成功概率的方法<br>
考虑如下代码:<br>
```c
int main(int argc, char **argv) {
    int echo_fd = open("/bin/echo", O_RDONLY);
    int fd = open(argv[1], O_WRONLY | O_CREAT, 0755);
    sendfile(fd, echo_fd, 0, 1024*1024);
    close(fd);
    execl(argv[1], argv[1], "SAFE", NULL);
}
//这段代码的空窗期比2.1提到的小得多，主要原因是/bin/sh执行时需要加载很多system call
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241213211831.png)
这概率小太多了，因此，我们需要提高空窗期的方法！<br>

### 2.2.1 方法一: nice<br>
`nice`命令和`nice system call`允许用户设置进程在`linux kernel`调度器中的优先级。<br>
`linux kernel scheduler`的优先级从高到低 为-20~19，优先级越高的进程，能够获得cpu资源的比例就越高<br>
`ionice`的用法和`nice`差不多，只不过它设置的是进程的io调度优先级，优先级从高到低是0~7，<br>
**这种方法的思路就是通过降低进程在系统中被执行的优先级，从而提高空窗期**<br>

用法如下:<br>
```
1. 启动一个terminal， 运行 while /bin/true; do cp -v catflag asdf; done
2. 再启动一个terminal，运行 for i in $(seq 1 2000); do nice -n 19 ./fs2 asdf; done | tee output
   运行sort output | uniq -c
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241214001123.png)
可以看到，概率还是命中次数差不多提升了一倍。<br>
`ionice也有差不多的效果`<br>
```
for i in $(seq 1 2000); do ionice -n 7 ./fs2 asdf; done | tee output
sort output | uniq -c
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241214102238.png)
两者结合，使用效果更佳:<br>
```
for i in $(seq 1 2000); do nice -n 19 ionice -n 7 ./fs2 asdf; done | tee output
sort output | uniq -c
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241214102358.png)

### 2.2.2 方法二: Path Complexity<br>
`filesystem race`涉及到文件系统访问，但是 不是所有的文件系统访问都是想等的<br>
```
cat my_file 比
cat a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/x/y/z/my_file
快得多！
因为内核需要花时间进入这些目录！
```
**这给了我们一个灵感：即可以传超级长路径来降低程序访问文件的速度,从而提高空窗期(需要注意，linux的路径限制最长为4096字节)**<br>
**另外，我们还可以通过符号链接来延长路径**<br>
```
for i in $(seq 1 2000); do ./fs2 a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/a/b/c/d/e/f/g/h/i/j/k/l/m/n/o/p/q/r/s/t/u/v/w/s/y/z/link/asdf ; done | tee output
sort output | uniq -c
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241215152509.png)
随着路径变长，成功的概率还能提高！<br>
这种`race condition`也导致了**TOCTOU(Time of check to Time of use)问题**<br>

## 2.3 mitigation<br>
这种`races condition`是有缓解措施的<br>
```
1. Safer programming practices (O_NOFOLLOW, mkstemp(), etc).
O_NOFOLLOW为open函数一个标志，表明不能打开符号链接的文件。
mkstemp() 是一种安全地创建临时文件的标准 C 函数。它会生成一个唯一的文件名并原子性地创建文件，同时避免了潜在的竞争条件（race condition）

2. Symlink protections in /tmp
a. root cannot follow symlinks in /tmp that are owned by other users
b. specifically made to prevent these sorts of issues
```

# 3. processes and threades<br>
## 3.1 进程和线程介绍<br>
进程有属于它自己的空间:<br>
```
1. virtual memory
   -stack
   -heap
   -etc
2. registers
3. file descriptors
4. Process ID
5. Security properties
   -uid
   -gid
   -seccomp rules
```
一个进程可以有多个线程（至少有一个，运行main函数），线程:<br>
```
1. 线程们会共享:
   -virtual memory
   -file descriptors
2. 线程们会独自拥有:
   -registers
   -stack
   -thread id
   -security properties(uid,gid,seccomp rules)
```

## 3.2 创建thread<br>

```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <stdlib.h>
#include <string.h>

void * pthread_main(int arg){
    printf("Thread %d,PID %d,TID %d,UID %d\n",arg,getpid(),gettid(),getuid());
}

int main(){
    printf("hello world!");
    pthread_t thread1, thread2;
    pthread_create(&thread1,NULL,pthread_main,1);
    pthread_create(&thread2,NULL,pthread_main,2);
    printf("Main thread: PID %d, TID %d, UID %d\n",getpid(),gettid(),getuid());
    pthread_join(thread1,NULL);//主进程等待thread1结束才会退出
    pthread_join(thread2,NULL);
}
```
由下图可以看到，**不同线程的执行顺序并不能得到保证。**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202024-12-17%20221301.png)

**事实上，线程创建是通过使用clone()系统调用实现的,clone()是fork()的继承者，它对于父子间共享什么内容 提供了更多的控制**<br>
`pthread_create()库函数，使用了clone()系统调用来创建一个子进程，这个子进程用来和父进程共享内存还有其他资源`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241218080654.png)
事实上上回说到的容器的创建也是用到了它。<br>

## 3.3 libc和syscall接口的差异<br>
libc提供的系统调用接口，和直接调用syscall，是有差异的。<br>
```
setuid: 
libc提供的setuid(a)会把所有线程的uid都设置为a；实际上syscall提供的setuid(a)只会把调用者线程的uid设置为a

exit：
libc提供的exit()实际上会调用exit_group() syscall，会结束所有线程；然而syscall提供的exit()只会结束调用者线程！
```
来看一个例子：<br>
```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <stdlib.h>
#include <string.h>


void * pthread_main(int arg){
    //if(arg == 1) syscall(105,1000);
    if(arg == 1) setuid(1000);
    else
        sleep(1);
    printf("Thread %d,PID %d,TID %d,UID %d\n",arg,getpid(),gettid(),getuid());
}

int main(){
    printf("hello world!\n");
    pthread_t thread1, thread2;
    pthread_create(&thread1,NULL,pthread_main,1);
    pthread_create(&thread2,NULL,pthread_main,2);
    sleep(1);
    printf("Main thread: PID %d, TID %d, UID %d\n",getpid(),gettid(),getuid());
    pthread_join(thread1,NULL);
    pthread_join(thread2,NULL);
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241218192528.png)
替换实际调用的函数后：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241218192617.png)


# 4. Races in memory<br>
内存是线程间共享的资源，内存条件竞争在多线程程序中十分泛滥。<br>
可以看一下实际代码:<br>
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
unsigned int size = 42;

void read_data(){
    char buffer[16];
    if(size<16){
        printf("Valid size! Enter payload up to %d bytes\n",size);
        printf("Read %d bytes!\n",read(0,buffer,size));
    }else{
        printf("Invalid size!\n",size);
    }
}
void *thread_allocator(int arg){
    while(1) read_data();
}

int main(){
    pthread_t allocator;
    pthread_create(&allocator,NULL,thread_allocator,0);
    while(size!=0) read(0,&size,1);
    exit(0);
}
```
这个代码的问题在于在线程当中，`size的check和size的实际使用之间是存在时间差的`<br>
利用方法：<br>
```shell
while true;do echo -ne "\x01\xff";done | ./pthread3
# -n表示不输出换行符，-e表示启用转义序列解析，即能识别\x形式
```
利用的核心思路是让线程执行如下操作：<br>
```
1. read(0,&size,1); 读入\x01
2. size<16校验通过
3. read(0,&size,1) 读入\xff
4. read(0,buffer,size)
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241219220040.png)

## 4.1 Double Fetch<br>
在linux kernel中也存在`race condition`，`double fetch`就是非常典型的一种:<br>
```c
// user_buffer: size(4bytes) + content
int check_safety(char *user_buffer, int maximum_size) {
    int size;
    copy_from_user(&size, user_buffer, sizeof(size));// 1 fetch
    return size <= maximum_size;
}

static long device_ioctl(struct file *file, unsigned int cmd, unsigned long user_buffer) {
    int size;
    char buffer[16];
    if (!check_safety(user_buffer, 16)) return;
    copy_from_user(&size, user_buffer, sizeof(size));// 2 fetch
    copy_from_user(buffer, user_buffer+sizeof(size), size);
}
```
**在第1次fetch，和第2次fetch间，如果user_buffer的内容被修改了，就能导致race condition**<br>

## 4.2 General data races<br>
`General date races`通常有着十分奇怪的影响。<br>
参考这个代码：<br>
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
unsigned int num = 0;
void *thread_main(int arg) {
    while (1) {
        num++;
        num--;
        if (num != 0) printf("NUM: %d\n", num);
    }
}

main() {
    pthread_t t1, t2;
    pthread_create(&t1, NULL, thread_main, 0);
    pthread_create(&t2, NULL, thread_main, 0);
    getchar();
    exit(0);
}
```
在这种情况下，`num`会随着运行时间而增加<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241221100920.png)
核心原因是在汇编层面，线程按照如下顺序执行了:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241221100956.png)

解决办法也是有的，就是加锁<br>
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>
pthread_mutex_t lock;
unsigned int num = 0;

void *thread_main(int arg) {
    while (1) {
	   pthread_mutex_lock(&lock);
      num++;
      num--;
      if (num != 0) printf("NUM: %d\n", num);
      pthread_mutex_unlock(&lock); //被锁保护区域叫做临界区
    }
}
main() {
   pthread_t t1, t2;
   pthread_create(&t1, NULL, thread_main, 0);
   pthread_create(&t2, NULL, thread_main, 0);
   getchar();
   exit(0);
}
```
这样即使你运行很久也不行。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241221103257.png)

# 5. Signals and reentrancy<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>