---
layout: post
tags: [pwn]
title: "sandboxing —— namespace"
author: wsxk
date: 2024-11-17
comments: true
---

- [1. 什么是 namespaces](#1-什么是-namespaces)
- [namespace 和 seccomp 的差异和关联](#namespace-和-seccomp-的差异和关联)


## 1. 什么是 namespaces<br>
`namespaces`的简介可以通过linux命令 `man namespaces`来了解<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241117192240.png)
总而言之，`namespaces`是linux kernel提供的用来隔离进程可用内核资源的机制。通过`namespaces`可以让进程只看到跟自己相关的一部分内核资源。<br>
从上述的`man`可以得知，当前`namespaces`只允许隔离6种类型的linux内核资源<br>
```
1. cgroup: 

2. IPC:

3. Network:

4. Mount:

5. PID:

6. UTS:
```

## namespace 和 seccomp 的差异和关联<br>
`namespace`用于限制进程可调用的系统资源，`seccomp`用于限制进程可以执行的系统调用；一定要说的话**seccomp的优先级大于namespace,比较namespace的调用本身依赖于执行系统调用**<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

