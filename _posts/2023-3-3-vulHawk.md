---
layout: post
tags: [iot]
title: "vulHawk"
author: wsxk
date: 2023-3-3
comments: true
---

- [前言](#前言)
- [论文研究意义](#论文研究意义)


## 前言<br>
这是来自论文`VulHawk: Cross-architecture Vulnerability Detection with Entropy-based Binary Code Search`的笔记<br>
ndss2023的paper<br>

## 论文研究意义<br>
二进制代码搜索通常被应用在寻找相似或相应的二进制函数，通过在一个大型的函数仓库中进行比对<br>
IOT firmware通常是二进制文件，来自于不同架构上的不同编译器经过不同的优化等级编译而来,所以二进制代码搜索通常不是那么好用。<br>
接下来是论文中喜闻乐见的研究现状描述场景<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230302145538.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230302145658.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230302145713.png)

然后这哥们也利用了我没听说过的基于熵的方法啊，图卷积网络啊（感觉是ai的），看起来十分牛逼，然后解决了如下问题:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230302151014.png)

这是一套用ai来帮助实现二进制函数相似性识别的工具:**首先把二进制代码转换成IR并且保留主要语义的同时进行简化，保留主要语义主要依赖2个事先训练好的模型**<br>
**其次使用熵信息流的方法识别二进制文件的文件环境，用基于熵的调解器对其进行调解，使得这些二进制文件的文件环境接近同一个文件，消解差异**<br>

总之，这不是一个二进制手子的理解范围，于是乎，跑路！<br>