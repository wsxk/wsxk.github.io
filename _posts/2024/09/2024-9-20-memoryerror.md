---
layout: post
tags: [pwn]
title: "memory errors"
date: 2024-9-20
author: wsxk
comments: true
---

- [1. introduction](#1-introduction)


## 1. introduction<br>
**内存破坏的起源思想：如果一个程序允许某人覆盖他们不应该覆盖的内存怎么办**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240914140009.png)
`Mainstream compiled languages`指的是编译语言(c,c++,etc)<br>
`VM-based languages`指的是解释语言(java,python,etc)<br>
编译语言带来的内存安全问题虽然严重，但是编译语言(c)运行的速度是最快的，所以到现在为止，`c/c++`仍然无处不在。<br>
目前，**想要保持速度，又能内存安全的尝试，就是Rust**（仍然努力中）<br>