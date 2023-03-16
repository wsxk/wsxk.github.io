---
layout: post
tags: [c++]
title: "c++ template& std::array"
author: wsxk
comments: true
date: 2023-3-14
---


- [template](#template)
- [std::array](#stdarray)
- [array 和 vector的区别](#array-和-vector的区别)


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


## std::array<br>
`std::array`位于`<array>`这个标准库中，**它是一个静态数组，一旦确认，不能更改大小**<br>
```c++
#include <iostream>
#include <array>
#include <vector>

template <int N>  //这里使用template是为了让传参时不必把5写上去
void PrintArray(std::array<int, N>& a) {
	for (int i = 0; i < a.size(); i++) {
		std::cout << a[i] << std::endl;
	}
}

int main() {
	std::array<int, 5> data;
	data[0] = 1;
	data[4] = 5;
	PrintArray(data);
}
```
**std::array在栈上分配，然而不分配内存来存储size**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230316172809.png)
因为 `_Size`是一个模板，所以在代码使用它后，他会实际填到这个函数中，也就说，这个函数返回一个常数，因此不需要内存来存储size.<br>

**std::array可以有边界检查，当然是可选的**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230316172614.png)

## array 和 vector的区别<br>
实际上区别如下:<br>
> 1. array是静态数组，vector是动态的
> 2. array在栈上分配，vector在heap中分配,因此用array会比vector快