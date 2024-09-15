---
layout: post
tags: [pwn]
title: "memory errors"
date: 2024-9-20
author: wsxk
comments: true
---

- [1. introduction](#1-introduction)
- [2. C high-level problems](#2-c-high-level-problems)
  - [2.1 Trusting the Developer](#21-trusting-the-developer)
  - [2.2 Mixing Control Information and Data](#22-mixing-control-information-and-data)
  - [2.3 Mixing Data and Metadata](#23-mixing-data-and-metadata)
  - [2.4 Initialization and Cleanup](#24-initialization-and-cleanup)


## 1. introduction<br>
**内存破坏的起源思想：如果一个程序允许某人覆盖他们不应该覆盖的内存怎么办**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240914140009.png)
`Mainstream compiled languages`指的是编译语言(c,c++,etc)<br>
`VM-based languages`指的是解释语言(java,python,etc)<br>
编译语言带来的内存安全问题虽然严重，但是编译语言(c)运行的速度是最快的，所以到现在为止，`c/c++`仍然无处不在。<br>
目前，**想要保持速度，又能内存安全的尝试，就是Rust**（仍然努力中）<br>

## 2. C high-level problems<br>
### 2.1 Trusting the Developer<br>
c语言是非常信任开发者的。<br>
```c
int a[3] = { 1, 2, 3 };
a[10] = 0x41;
// no problem!
```
像python语言，这种写法不会通过。<br>
```python
>>> a = [ 1, 2, 3 ]
>>> print a[10] = 0x41;
IndexError: list index out of range
```

### 2.2 Mixing Control Information and Data<br>
很好理解，比如在栈结构中，函数的返回地址与用户数据是相邻的。<br>

### 2.3 Mixing Data and Metadata<br>
主要体现在字符串上。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240915091327.png)

### 2.4 Initialization and Cleanup<br>
c不会帮你自动初始化一个变量的值，当然，也不会帮你清理值。<br>
```c
int a; //a的值未知，取决于栈中的值，不会初始化

char * b = malloc(20);
free(b); // free后，内存中的值也不会自动清除
```