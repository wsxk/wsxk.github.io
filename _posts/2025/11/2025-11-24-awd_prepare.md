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


## 2.4 流量检测<br>

## 2.5 后门查杀<br>

## 2.6 挖洞<br>

## 2.7 修洞<br>

## 2.8 exp编写<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>