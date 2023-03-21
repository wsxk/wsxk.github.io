---
layout: post
tags: [c++]
title: "c++多维数组& std::sort &类型双关 & union"
author: wsxk
comments: true
date: 2023-3-20
---

- [多维数组](#多维数组)
	- [多维数组优化实现](#多维数组优化实现)
- [std::sort](#stdsort)
- [类型双关](#类型双关)
- [union](#union)


## 多维数组<br>
多维数组是个很抽象的东西<br>
```c++
#include <iostream>

int main() {
	int** a2d = new int* [50];
	for (int i = 0; i < 50; i++) {
		a2d[i] = new int[50];
	}

	int*** a3d = new int** [50];
	for (int i = 0; i < 50; i++) {
		a3d[i] = new int* [50];
		for (int j = 0; j < 50; j++) {
			a3d[i][j] = new int[50];
		}
	}
}
```
值得注意的是，因为没有 `delete[][]`这种字符，所以释放内存时需要十分注意<br>
```c++
#include <iostream>

int main() {
	int** a2d = new int* [50];
	for (int i = 0; i < 50; i++) {
		a2d[i] = new int[50];
	}

	int*** a3d = new int** [50];
	for (int i = 0; i < 50; i++) {
		a3d[i] = new int* [50];
		for (int j = 0; j < 50; j++) {
			a3d[i][j] = new int[50];
		}
	}
	
	for (int i = 0; i < 50; i++) {
		delete[] a2d[i];
	}
	delete[] a2d;

	for (int i= 0; i < 50; i++) {
		for (int j = 0; j < 50; j++) {
			delete[] a3d[i][j];
		}
		delete[] a3d[i];
	}
	delete[] a3d;
}
```

### 多维数组优化实现<br>
转化为一维数组，减少`cache miss`,提速<br>
```c++
#include <iostream>
#include <chrono>

class Timer {
	std::chrono::steady_clock::time_point start, end;
	std::chrono::duration<float> duration;
public:
	Timer() {
		start = std::chrono::high_resolution_clock::now();
	}
	~Timer() {
		end = std::chrono::high_resolution_clock::now();
		duration = end - start;
		float ms = duration.count() * 1000.0f;
		std::cout << "Timer took " << ms << "ms" << std::endl;
	}
};

void function1() {
	int** a2d = new int* [50];
	for (int i = 0; i < 50; i++) {
		a2d[i] = new int[50];
	}
	Timer time;
	for (int i = 0; i < 50; i++) {
		for (int j = 0; j < 50; j++) {
			a2d[j][i] = 2;
		}
	}
}

void function2() {
	int* array = new int[50 * 50];
	Timer time2 ;
	for (int i = 0; i < 50; i++) {
		for (int j = 0; j < 50; j++) {
			array[i * 50 + j] = 2;
		}
	}
}

int main() {
	function1();
	function2();
}
```

## std::sort<br>
```c++
#include <iostream>
#include <algorithm>
#include <vector>
#include <functional>

int main() {
	std::vector<int> values = { 1,3,5,4,2 };
	//std::sort(values.begin(), values.end());
	//std::sort(values.begin(), values.end(),std::greater<int>());
	std::sort(values.begin(), values.end(), [](int a, int b) {return a > b; });
	for (int value : values) {
		std::cout << value << std::endl;
	}
}
```

## 类型双关<br>
其实类型双关指的是，一块内存用不同类型来解释<br>
看个例子:<br>
```c++
#include <iostream>
int main() {
	int a = 50;
	double value = a;
	std::cout << value << std::endl;
	return 0;
}
```
value之所以是0是因为c++编译器做了隐式转换，把`a(50)`转化成了浮点数字50<br>
用下面这个代码就可以看出端倪<br>
```c++
#include <iostream>
int main() {
	int a = 50;
	double value = *(double *) & a;
	std::cout << value << std::endl;
	return 0;
}
```
此时，value不等于50，而是一个神秘数字，**原因是浮点数和整数在机器中有不同的表达形式，然而这种情况下，其实a和value所指向的内存的二进制表示是相同的**<br>
注意在这种情况下有一个隐患，就是int是4字节，double是8字节。<br>
所以如果是下面这种形式，程序有可能崩溃:<br>
```c++
#include <iostream>
int main() {
	int a = 50;
	double *value = (double *) & a;
	*value = 0.0;
	std::cout << value << std::endl;
	return 0;
}
```

## union<br>
union通常和类型双关结合起来，并且经常以`匿名`的形式出现<br>
```c++
#include <iostream>

struct Vector2 {
	float x, y;
};

struct Vector4 {
	union 
	{
		struct {
			float x, y, z, w;
		};

		struct {
			Vector2 a, b;
		};
	};
};

void PrintVector2(const Vector2& vector) {
	std::cout << vector.x << " " << vector.y << std::endl;
}

int main() {
	Vector4 a = { 1.0f,2.0f,3.0f,4.0f };
	PrintVector2(a.a);
	PrintVector2(a.b);
	std::cout << "-------------------------------" << std::endl;
	a.x = 3.0f;
	PrintVector2(a.a);
	PrintVector2(a.b);

}
```