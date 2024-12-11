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


# 1. 什么是race condition<br>
在古早时期，CPU是单核的，但是你想在这颗CPU上运行多个进程，这意味着:<br>
```
用户感受到两个进程同时运行，但是实际上
在某个时刻内，只有一个进程能被执行
只不过用户(人)感受不到这个变化
```
现代，CPU是多核的，但是:<br>
```
1. 进程数量还是多于核数
2. 内核因此还是需要决定什么时候调度哪些进程
3. 存储控制器的通道有限（四通道），存储媒介通道有限，网络通信是单通道的
```
这些限制都导致计算机无法同时运行所有进程。<br>
`race condition`出现的核心要旨是**计算设备的瓶颈导致并发的时间至少有一部分需要被序列化**<br>
**通常，没有隐式依赖或者程序显式的努力，执行顺序只能在进程内(一个线程)得到保证**<br>
这也导致一个问题，如果进程A和B都检查文件C的内容(check),符合条件就更新文件C(change)；然后进程A和B按照如下顺序被执行:<br>
```
1. A check #通过
2. B check #通过
3. A change # A改变了C的内容
4. B change # 讲道理此时不应该执行B，但是B还是能被执行，导致C的内容被改变了2次
```
这就是`race condition`!<br>
**要想利用race condition，我们需要在应用的薄弱时期，精细地影响其状态才行**<br>

# 2. races in filesystem<br>
 

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>