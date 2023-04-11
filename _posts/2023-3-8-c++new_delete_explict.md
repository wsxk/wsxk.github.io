---
layout: post
tags: [c++]
title: "c++ new & delete & explict"
date: 2023-3-8
author: wsxk
comments: true
---

- [new/delete](#newdelete)
- [new/delete实际上做了什么](#newdelete实际上做了什么)
- [explict关键字](#explict关键字)
	- [如何避免意外的隐式转换](#如何避免意外的隐式转换)


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


## explict关键字<br>
直接先来看代码
```c++
#include <iostream>
#include <string>

class Entity {
public:
	std::string m_name;
	int m_age;
	Entity(const std::string &name)
		:m_name(name),m_age(-1) {
	}
	Entity(int age)
		:m_name("unknown"), m_age(age) {
	}
};

void print_entity(Entity& entity) {
	std::cout << entity.m_name << std::endl;
}
int main() {
	Entity a = 22;//？神马东西
	print_entity(a);
}
```
其实咱们可以发现，它还是能正常运行的。原因在于c++编译器自动对其做了隐式转换（`implict convert`）<br>
然后还有个问题<br>
```c++
#include <iostream>
#include <string>

class Entity {
public:
	std::string m_name;
	int m_age;
	Entity(const std::string &name)
		:m_name(name),m_age(-1) {
	}
	Entity(int age)
		:m_name("unknown"), m_age(age) {
	}
};

void print_entity(Entity& entity) {
	std::cout << entity.m_name << std::endl;
}
int main() {
	Entity a = 22;
	print_entity(a);
	Entity b = "wsxk";//无法通过编译
	print_entity(b);
}
```
`implict convert`只能隐式转换一次，"wsxk"是一个`const char *`类型，需要转变为`std::string`类型，然后再转变为`Entity`类型，其中隐式转换了2次，这是编译器不允许的.<br>

### 如何避免意外的隐式转换<br>
利用`explict`关键字<br>
```c++
#include <iostream>
#include <string>

class Entity {
public:
	std::string m_name;
	int m_age;
	explicit Entity(const std::string &name)
		:m_name(name),m_age(-1) {
	}
	explicit Entity(int age)
		:m_name("unknown"), m_age(age) {
	}
};

void print_entity(Entity& entity) {
	std::cout << entity.m_name << std::endl;
}
int main() {
	Entity a = 22; //报错
	print_entity(a);
	Entity b = "wsxk";//报错
	print_entity(b);
}
```