---
layout: post
tags: [c++]
title: "c++ 函数指针 & lambda & namespace"
author: wsxk
comments: true
date: 2023-3-17
---

- [函数指针](#函数指针)
- [lambda函数](#lambda函数)
	- [lambda作为迭代器的判断条件](#lambda作为迭代器的判断条件)
- [namespace](#namespace)


## 函数指针<br>
```c++
#include <iostream>

void Hello_world() {
	std::cout << "hello world!" << std::endl;
}

int main() {
	typedef void(*HelloWorld)();
	auto func = Hello_world;
	void (*p)() = &Hello_world;
	func();
	p();
	HelloWorld c = Hello_world;
	c();
}
```

函数指针还可以这样用
```c++
#include <iostream>
#include <vector>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,void(*func)(int)){
	for (int value : values) {
		func(value);
	}
}

int main() {
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, Hello_world);
}
```

## lambda函数<br>
lambda函数其实是一种匿名函数，我们写一个简短的函数，又不会在其他地方用，就适合用它。<br>
上面的内容可以改成如下形式<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,void(*func)(int)){
	for (int value : values) {
		func(value);
	}
}

int main() {
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [](int value) {std::cout << value << std::endl; });
}
```
lambda的更详细信息可以去[https://en.cppreference.com/w/](https://en.cppreference.com/w/)看<br>
`[]`用于捕获其他变量，常用的有`=（值传入所有变量）` `&（引用传入所有变量）`<br>
`()`用于传参<br>
`{}`就是一个普通函数体<br>

```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,std::function<void(int)>func){
	for (int value : values) {
		func(value);
	}
}

int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [=](int value) {std::cout << value << std::endl; std::cout << a << std::endl; });
}
```
但是注意的是，通过值传递是无法修改变量的，在函数体内也不能做类似于`a=5`的操作<br>
添加`mutable`关键字后就可以了。<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,std::function<void(int)>func){
	for (int value : values) {
		func(value);
	}
}

int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [=](int value) mutable {a = 5; std::cout << value << std::endl; std::cout << a << std::endl; });
	std::cout << a << std::endl; //当然不改变值传递的本质，仍然为10
}
```
通过引用传递的是会修改的<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,std::function<void(int)>func){
	for (int value : values) {
		func(value);
	}
}

int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [&](int value) {a = 5; std::cout << value << std::endl; std::cout << a << std::endl; });
	std::cout << a << std::endl; //是5
}
```

### lambda作为迭代器的判断条件<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>
int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	auto it = std::find_if(values.begin(), values.end(), [](int value) {return value > 3; });
	std::cout << it[1] << std::endl;
}
```

## namespace<br>
`namespace`其实就起一个标识函数的作用<br>
```c++
#include<iostream>
#include <string>
#include <algorithm>

namespace apple {
	void print(const std::string& text) {
		std::cout << text << std::endl;
	}
}

namespace orange {
	void print(const char* text) {
		std::string temp = text;
		std::reverse(temp.begin(), temp.end());
		std::cout << temp << std::endl;
	}
}

int main() {
	apple::print("wsxk");
	orange::print("wsxk");
	using namespace apple;
	namespace a = apple;
	print("wsxk");
	a::print("wsxk");
}
```
