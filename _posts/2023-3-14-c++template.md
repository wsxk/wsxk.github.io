---
layout: post
tags: [c++]
title: "c++ template"
author: wsxk
comments: true
date: 2023-3-14
---


- [template](#template)


## template<br>
template是c++一个强大的特性<br>
它可以减少我们的代码重复，比如书写一个打印不同类型值的函数的时候<br>
```c++
#include <iostream>
#include <string>

template<typename T>
void print(T value) {
	std::cout << value << std::endl;
}

int main() {
	print(1);
	print<int>(0.5);
	print("hello");
}
```

**template实际上做的是，当遇到一个类型时，会自动把该类型填入到T中，并生成相应的代码**，如下:
```c++
#include <iostream>
#include <string>

template<typename T>
void print(T value) {
	std::cout << value << std::endl;
}

void print(int value) {
	std::cout << value << std::endl;
}
void print(std::string value) {
	std::cout << value << std::endl;
}
void print(float value) {
	std::cout << value << std::endl;
}

int main() {
	print(1);
	print<int>(0.5); // <int>可以显式得指定模板的类型
    print(0.5f);
	print("hello");
}
```

template还能做到很多有趣的事情，比如自动填写类<br>
比如我们想要一个可以动态调整数组大小的类，我们可以这样用<br>
```c++
#include <iostream>
#include <string>

template<int N>
class Array {
private:
	int m_array[N];
public:
	int GetSize() { return N; }
};

int main() {
	Array<5> a;
	std::cout << a.GetSize() << std::endl;
}
```

更进一步，如果我们想要数组存储的不一定是int类型的整数，还可以是其他类型时，我们可以这样用<br>
```c++
#include <iostream>
#include <string>

template<typename T,int N>
class Array {
private:
	T m_array[N];
public:
	int GetSize() { return N; }
};

int main() {
	Array<float,5> a;
	std::cout << a.GetSize() << std::endl;
}
```
十分高级<br>


