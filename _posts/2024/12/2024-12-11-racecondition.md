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
- [3. processes and threades](#3-processes-and-threades)


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
这概率小太多了，因此，我们需要提高成功率的方法！<br>

### 2.2.1 方法一: nice<br>
`nice`命令和`nice system call`允许用户设置进程在`linux kernel`调度器中的优先级。<br>
`linux kernel scheduler`的优先级从高到低 为-20~19，优先级越高的进程，能够获得cpu资源的比例就越高<br>
`ionice`的用法和`nice`差不多，只不过它设置的是进程的io调度优先级<br>

用法如下:<br>
```
1. 启动一个terminal， 运行 while /bin/true; do cp -v catflag asdf; done
2. 再启动一个terminal，运行 for i in $(seq 1 2000); do nice -n 19 ./fs2 asdf; done | tee output
   运行sort output | uniq -c
```

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241214001123.png)
可以看到，概率还是命中次数差不多提升了一倍。<br>



# 3. processes and threades<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>