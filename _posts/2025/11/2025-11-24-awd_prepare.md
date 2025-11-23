---
layout: post
tags: [awd]
title: "awd 赛前准备"
author: wsxk
date: 2025-11-24
comments: true
---

- [1. 环境准备](#1-环境准备)
  - [1.1 虚拟机](#11-虚拟机)
- [2. awd核心步骤](#2-awd核心步骤)
  - [2.1 ssh登录改密码](#21-ssh登录改密码)
  - [2.2 文件备份](#22-文件备份)
  - [2.3 通防](#23-通防)
  - [2.4 流量检测](#24-流量检测)
  - [2.5 后门查杀](#25-后门查杀)
  - [2.6 挖洞](#26-挖洞)
  - [2.7 修洞](#27-修洞)
  - [2.8 exp编写](#28-exp编写)


# 1. 环境准备<br>
## 1.1 虚拟机<br>
[pwn 24.04环境搭建保姆级教程](https://blog.csdn.net/j284886202/article/details/139352067)<br>
总的来说，要装以下东西:<br>
```
1. ubuntu 24.04 虚拟机
2. vim（编辑器）
3. gcc（编译器）
4. git
5. python3-pip
6. pwntools
7. pwndbg
8. one_gadget
9. glibc-all-in-one
10. patchelf
11. cmake
12. wabt、wasmtime
13. curl
14. qemu
15. seccomp-tools
16. seccomp
17. net-tools
18. docker
19. docker-compose
20. VS code及其插件
```

# 2. awd核心步骤<br>
这回不知道为什么，awd开启了无限制击打游戏；即出题方默许了删站、种马等一劳永逸的恶心人做法，没办法，只能开启斗法时代了。<br>

## 2.1 ssh登录改密码<br>
登录后光速改密码，以防密码是弱密码直接被爆了。<br>

```
ssh user@192.168.1.100 -p 2222
#登录后执行：
passwd
```

## 2.2 文件备份<br>
本质是先把题目都拷贝一份下来，防止有人恶意搅屎把你的题目全删喽或者搞崩喽。<br>
```
scp local.txt user@remote:/path/to/dir/ #拷贝本地文件到远程
scp user@remote:/path/to/file.txt /local/dir/ #拷贝远程文件到本地
scp -r user@remote:/path/to/remotedir/ ./localdir/ #拷贝远程目录到本地
```

## 2.3 通防<br>
暂时没用上，核心思路是添加一个禁止`sys_execve`执行的系统调用:<br>
```asm
45 31 D2                      xor     r10d, r10d
45 31 C0                      xor     r8d, r8d
6A 26                         push    26h ; '&'
5F                            pop     rdi
31 D2                         xor     edx, edx
6A 01                         push    1
5E                            pop     rsi
31 C0                         xor     eax, eax
B0 9D                         mov     al, 9Dh
0F 05                         syscall                                 ; LINUX - sys_prctl
6A 06                         push    6
48 B8 06 00 00 00 00 00 FF 7F mov     rax, 7FFF000000000006h
50                            push    rax
48 B8 15 00 01 00 3B 00 00 00 mov     rax, 3B00010015h
50                            push    rax
6A 20                         push    20h ; ' '
54                            push    rsp
6A 04                         push    4
49 89 CA                      mov     r10, rcx
6A 16                         push    16h
5F                            pop     rdi
48 89 E2                      mov     rdx, rsp
6A 02                         push    2
5E                            pop     rsi
31 C0                         xor     eax, eax
B0 9D                         mov     al, 9Dh
0F 05                         syscall                                 ; LINUX - sys_prctl
48 83 C4 30                   add     rsp, 30h
```


## 2.4 流量检测<br>
场景：可通过ssh访问靶机（以ctf用户权限登录进行维护换题），可通过nc访问靶机999端口（开放题目）。<br>
限制**在靶机中，题目通过docker（docker内部使用xinted）端口映射到靶机999端口。靶机中的ctf用户被隔离，无法观测docker内环境。**<br>
todo: 如何进行流量抓取。<br>
思路一：替换题目，选手攻击时将流量转发到靶机，靶机起程序流量转发到选手本地。<br>
思路二：替换题目，选手攻击时将流量保存到内存，题目植入后门，我们登录后把内存取出。<br>

## 2.5 后门查杀<br>
暂时没用上

## 2.6 挖洞<br>
以二进制而言，主要还是看你逆向分析的速度。<br>

## 2.7 修洞<br>
[https://wsxk.github.io/awdpatch/](https://wsxk.github.io/awdpatch/)<br>
遇到checker比较严格的场景：**最多允许修改100字节，且只允许修改代码段区间**，这种场景下比较考验选手的汇编功底，一般情况需要查看汇编的上下文，寻找可以移除的汇编指令。<br>

## 2.8 exp编写<br>
跟平时做pwn题没什么区别。一定要说的话，就是不知道对手到底补了什么东西有点烦。<br>
todo：常见exp模板准备。<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>