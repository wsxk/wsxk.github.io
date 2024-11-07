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
  - [2.2 Access Control Matrix 实施策略](#22-access-control-matrix-实施策略)
- [3. POSIX的ACL](#3-posix的acl)
- [4. 访问控制的类型](#4-访问控制的类型)
  - [4.1 Mandatory Access Control讲解](#41-mandatory-access-control讲解)

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## 前言<br>
访问控制其实和linux中遇到的文件系统权限，apparmor是息息相关的
自我宣传一下.jpg 😄<br>
**这个章节讲述的是linux中访问控制策略实施的原型**<br>
各位如果感兴趣，还可以看看本人的其他章节~<br>
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
Subjects are processes（进程,可以说是域，即属主，属主域，其他）
p, q

Files are objects（文件）
f, g

Rights (read, write, execute, append, own)
r, w, x, a, o 
```

### 2.1 Access Control Matrix<br>
访问控制矩阵，**subjects作为行，subjects+objects作为列，Rights表示subject可以对subject/object做的动作**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240705214741.png)

### 2.2 Access Control Matrix 实施策略<br>
访问控制矩阵的每一列都会存储在`objects（文件）`里,这一列被称作**ACLs（Access Control Lists）**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240706172138.png)
具体到linux系统中，即`ls -l` 显示出的权限<br>

另一方面，访问控制矩阵的每一行都会存储在`subjects`中<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240706173215.png)
具体到linux系统中，即`getcap /usr/bin/ping`可以查看的内容<br>

`ACL`和`capabilities`的区别<br>
```
1、 ACL要求认证（即你是谁），CAP不需要，所以CAP不能被仿造且传播时必须要被良好控制

2、CAP提供了最细粒度的权限控制，特别是被创建来执行特定任务的短期进程...但是实际上不可能记录进程对每个文件的权限，所以linux中CAP主要限制的是内核的资源

3、ACL对文件更好用，CAP对进程更好用
撤销也是如此~
```

## 3. POSIX的ACL<br>
用`12bits`来表示各个文件的ACL<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240707110723.png)
```
--- SUID SGID Sticky-bit
--- r    w    x     owner
--- r    w    x     file's group
--- r    w    x     other
```

可以通过`cat /etc/passwd`查看用户基本信息<br>
查询到的信息一般如下所示<br>
```
root:x:0:0:root:/root:/bin/bash

:表示分隔符
1. root : 表示用户名
2. x    ：表示密码，x是占位符，如果x在就说明密码被存在另外一个文件里，一般是/etc/shadow
3. 0    ：用户id，即uid
4. 0    ：组id，即gid
5. root ：用户描述字段，告诉你这个用户是干啥的
6. /root：用户的主要目录在哪里
7. /bin/bash：用户登录时使用的shell程序
```

现在查看一下`/etc/shadow`中的内容<br>
```
root:!:19531:0:99999:7:::

1. root ： 表示用户名
2. !    ： 加密后的密码，如果是！或* ，说明无法用这个账户登录
3. 19531： 上次密码更改的日期，这是自1970年1月1日以来的天数，表示上次密码更改的日期。可以使用这个数字计算具体的日期。19531 表示 2023年7月16日。
4. 0    ： 密码最小使用天数，0表示没有限制
5. 99999： 密码最大使用天数，99999基本表示永不过期了~
6. 7    ： 密码警告期，密码到期前7天发出警告，提示用户改天数
7.      ： 密码不活动期，用户密码过期后账号被锁定前的非活动天数。空字段表示没有定义
8.      ： 账号到期日期，自1970年1月一日以来的天数，没有内容表示账号永不过期
9.      ： 保留字段
```

## 4. 访问控制的类型<br>
```
Discretionary Access Control:自主访问控制
即object的拥有者可以决定谁能访问这个object

Mandatory Access Control： 强制访问控制
系统决定谁能访问objcet

Originator Controlled Access Control： 基于原创者的访问控制
object的创始人决定其他人对该object的权限
目前还没有有效的实现，因为目前的访问控制都是针对实体，而它是针对实体中的信息

Role Based Access Control (RBAC)：基于角色的访问控制
用户的权限由其角色控制，并且每个主体在任何时刻只能有一个角色

Attribute Based Access Control (ABAC)：基于属性的访问控制
用户会有一些属性（id，年龄，身份，etc），而访问控制策略就是各种属性的复杂布尔表达式
```
**在自主访问控制和强制访问控制都存在的系统中，一般先进行强制访问控制，然后再进行自主访问控制**<br>

### 4.1 Mandatory Access Control讲解<br>
现实中MAC会如何实现呢?<br>
军事机密等级，从上到下分别是`top secret、secret、confidential、unclassfied`
在MAC中，也有类似的等级，接下来说定义：<br>
```
L(S) = ls is the security clearance of subject S

L(O) = lO is the security classification of object O

For all security classifications li,i=0, …, k-1, li < li+1 

总之，每个`subject和object都有一个等级`。比较规则如下:

Simple-Security Condition (preliminary version)
S can read O if lO ≤ lS 

*-Property (preliminary version)
S can write O if lS ≤ lO

如果s的等级比o大，就可以读o的内容，这个很好理解
如果s的等级比o小，就可以写o的内容，这里主要是因为：
如果s的等级比o大就可以写o的内容的话，s可以把o的内容读出，然后写入其他低等级的文件中，导致信息向下泄露
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240709213852.png)
但是这种机制太粗粒度了，又有了新的分类：<br>

```
The security level (L, C) dominates the security level (L’, C’) if L’ ≤ L and C’ ⊆C

实践起来就是：
S can read O iff S dom O
即 LO ≤ LS and CO ⊆ CS

*-Property
S can write to O iff O dom S
即 LS ≤ LO and CS ⊆ CO
```
这就是著名的**Bell-LaPadula Model（贝尔-拉帕杜拉模型）**<br>
