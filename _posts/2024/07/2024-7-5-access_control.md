---
layout: post
tags: [linux]
title: "访问控制"
author: wsxk
comments: true
date: 2024-7-5
---

- [前言](#前言)
- [1. authorization VS authentication](#1-authorization-vs-authentication)
- [2. Modeling Access Control](#2-modeling-access-control)
  - [2.1 Access Control Matrix](#21-access-control-matrix)

## 前言<br>
访问控制其实和linux中遇到的文件系统权限，apparmor是息息相关的
自我宣传一下.jpg 😄<br>
[linux 文件/目录 权限管理](https://wsxk.github.io/linux%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86/)<br>
[AppArmor 访问控制](https://wsxk.github.io/apparmor%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6/)<br>
当然不看也可以~<br>

## 1. authorization VS authentication<br>
`authorization(授权)`表达的是你能够做什么<br>
`authentication(认证)`表达的是你是谁<br>

## 2. Modeling Access Control<br>
首先需要对访问控制进行建模
```
Subjects S
Things in the system that can act
即执行者

Objects O
Assets or objects in the system (acted upon)
资产/物体，可以被执行

Rights R
What can the subject do to the object?
即S可以对O做的事情，即权限
```

在一个简单的`Unix Model`中，系统中各个物体映射到访问控制模型的结果如下:<br>
```
Subjects are processes（进程）
p, q

Files are objects（文件）
f, g

Rights (read, write, execute, append, own)
r, w, x, a, o 
```

### 2.1 Access Control Matrix<br>
访问控制矩阵，**subjects作为行，subjects+objects作为列，Rights表示subject可以对subject/object做的动作**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240705214741.png)
