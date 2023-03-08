---
layout: post
tags: [c++]
title: "c++ new&delete"
date: 2023-3-8
author: wsxk
comments: true
---

- [new/delete](#newdelete)
- [new/delete实际上做了什么](#newdelete实际上做了什么)


## new/delete<br>
`new` 和 `delete`是c++中的用于在`heap`上申请内存空间和释放空间的关键字。<br>

直接上例子
```c++
#include <iostream>
#include <string>
int main() {
	int* a = new int(5);
	std::cout << *a << std::endl;
	delete a;
	int* b = new int[50];
	delete[] b;
}
```
`delete[]`用于当你申请的是一个数组时。<br>

## new/delete实际上做了什么<br>
new实际上调用了`malloc`函数，然后再调用类的构造函数。<br>
delete实际上调用了类的析构函数，然后调用了`free`函数