---
layout: post
tags: [c++]
title: "c++多维数组& std::sort &类型双关 & union & 虚析构函数 & 类型转换"
author: wsxk
comments: true
date: 2023-3-20
---

- [多维数组](#多维数组)
	- [多维数组优化实现](#多维数组优化实现)
- [std::sort](#stdsort)
- [类型双关](#类型双关)
- [union](#union)
- [虚析构函数](#虚析构函数)
	- [解决办法](#解决办法)
- [类型转换](#类型转换)
	- [c type cast](#c-type-cast)
	- [c++ type cast](#c-type-cast-1)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

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

## 虚析构函数<br>
虚函数大家都已经知道是什么情况了，现在可以看看虚析构函数。<br>
请测试如下代码:
```c++
#include <iostream>

class Base {
public:
	Base() {
		std::cout << "Base Constructior\n";
	}
	~Base() {
		std::cout << "Base Destructior\n";
	}
};

class Derived:public Base {
public:
	Derived() {
		std::cout << "Derived Constructior\n";
	}
	~Derived() {
		std::cout << "Derived Destructior\n";
	}
};

int main() {
	Base* base = new Base();
	delete base;
	std::cout << "------------------------------\n";
	Derived *derived = new Derived();
	delete derived;
	std::cout << "------------------------------\n";
	Base* poly = new Derived();
	delete poly;
	std::cout << "------------------------------\n";
}
```
我们会发现，poly析构函数只调用了一个<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230322130833.png)
在当前情况下可能没有问题，但是稍微修改一下就可能造成内存泄漏。<br>
```c++
#include <iostream>

class Base {
public:
	Base() {
		std::cout << "Base Constructior\n";
	}
	~Base() {
		std::cout << "Base Destructior\n";
	}
};

class Derived:public Base {
public:
	Derived() {
		m_alloc = new int[5];
		std::cout << "Derived Constructior\n";
	}
	~Derived() {
		delete[] m_alloc; //没有被调用，就无法释放内存
		std::cout << "Derived Destructior\n";
	}
private:
	int* m_alloc;
};

int main() {
	Base* base = new Base();
	delete base;
	std::cout << "------------------------------\n";
	Derived *derived = new Derived();
	delete derived;
	std::cout << "------------------------------\n";
	Base* poly = new Derived();
	delete poly;
	std::cout << "------------------------------\n";
}
```
### 解决办法<br>
在析构函数前加`virtual`关键字<br>
**创建基类时永远在析构函数前添加virtual关键字**<br>
```c++
#include <iostream>

class Base {
public:
	Base() {
		std::cout << "Base Constructior\n";
	}
	virtual ~Base() {
		std::cout << "Base Destructior\n";
	}
};

class Derived:public Base {
public:
	Derived() {
		m_alloc = new int[5];
		std::cout << "Derived Constructior\n";
	}
	~Derived() {
		delete[] m_alloc;
		std::cout << "Derived Destructior\n";
	}
private:
	int* m_alloc;
};

class A : public Derived {
public:
	A() {
		std::cout << "A Constructior\n";
	}
	virtual ~A() {
		std::cout << "A Destructior\n";
	}
};

int main() {
	Base* base = new Base();
	delete base;
	std::cout << "------------------------------\n";
	Derived *derived = new Derived();
	delete derived;
	std::cout << "------------------------------\n";
	Base* poly = new Derived();
	delete poly;
	std::cout << "------------------------------\n";
	Base* class_a = new A();
	delete class_a;
	std::cout << "------------------------------\n";
}
```

## 类型转换<br>
至少有两种类型的类型转换,`c风格`的和`c++风格`的。<br>
值得一提的是，`c++风格`的并不比`c风格`的干更多的事情，只是让转换更安全（当然更耗时）<br>
### c type cast<br>
```c++
#include <iostream>
int main(){
	double value = 5.25;
	double a = int(value) + 5.3;
}
```

### c++ type cast<br>
**再次强调：c++ 类型转换只是一个`semantic sugar`**<br>
**并且，这些cast可能需要实践来学习，单纯了解后可能没什么用**<br>
c++ type cast共有四种类型<br>
`1. static_cast`：<br>
static_cast是最常用的类型转换运算符。它可以用于基本数据类型之间的转换，如整数、浮点数等，还可以用于类层次结构中的上行和下行转换。上行转换是指从派生类到基类的转换，这种转换通常是安全的；下行转换是指从基类到派生类的转换，这种转换可能不安全，因为基类对象可能不包含派生类对象所需的全部信息。static_cast在编译时进行类型检查。<br>
```c++
#include <iostream>
class Base {
public:
	Base() {
		std::cout << "Base Constructior\n";
	}
	virtual ~Base() {
		std::cout << "Base Destructior\n";
	}
};
class Derived :public Base {
public:
	Derived() {
		std::cout << "Derived Constructior\n";
	}
	~Derived() {
		std::cout << "Derived Destructior\n";
	}
};
class AnotherClass : public Base {
public:
	AnotherClass() {
		std::cout << "A Constructior\n";
	}
	virtual ~AnotherClass() {
		std::cout << "A Destructior\n";
	}
};

int main() {
	double a = 5.35;
	double b = static_cast<int>(a);
	std::cout << b << std::endl;

	Derived* derived = new Derived();
	Base* base = static_cast<Base*>(derived);
	//Base* base = new Base();
	//Derived* c = static_cast<Derived*>(base); //不安全，因为不能保证基类拥有派生类的所有对象
}
```

`2. dynamic_cast`：<br>
dynamic_cast主要用于处理多态情况下的类型转换。它用于在类层次结构中进行安全的下行转换，即从基类指针或引用转换为派生类指针或引用。dynamic_cast在运行时执行类型检查，如果转换失败（例如，试图将基类对象转换为与其不相关的派生类对象），则返回空指针（针对指针类型）或抛出异常（针对引用类型）。<br>
```c++
#include <iostream>
class Base {
public:
	Base() {
		std::cout << "Base Constructior\n";
	}
	virtual ~Base() {
		std::cout << "Base Destructior\n";
	}
};
class Derived :public Base {
public:
	Derived() {
		std::cout << "Derived Constructior\n";
	}
	~Derived() {
		std::cout << "Derived Destructior\n";
	}
};
class AnotherClass : public Base {
public:
	AnotherClass() {
		std::cout << "A Constructior\n";
	}
	virtual ~AnotherClass() {
		std::cout << "A Destructior\n";
	}
};

int main() {
	double a = 5.35;
	double b = static_cast<int>(a);
	std::cout << b << std::endl;

	//Derived* derived = new Derived();
	//Base* base = dynamic_cast<Base*>(derived);
	Base* base = new Base();
	Derived* c = dynamic_cast<Derived*>(base); //不安全，因为不能保证基类拥有派生类的所有对象 //实际调试时会发现，这个值指针值是0，因为不安全
	return 0;
}
```
值得一提的是，`dynamic_cast`，是基于虚函数表来实现检测的，所以必须要有它才行。<br>


`3. reinterpret_cast`：<br>
reinterpret_cast用于执行低级别的、不安全的类型转换。它可以用于将一个指针类型转换为另一个不相关的指针类型，或者将指针类型转换为整数类型，反之亦然。此外，它还可以用于将函数指针转换为其他类型的函数指针。reinterpret_cast的结果是完全依赖于编译器的，因此在使用时要非常小心，以避免未定义的行为。<br>
**其实reinterpret_cast比较接近于一个内存当不同类型用**<br>
```c++
#include <iostream>

int main() {
    int i = 42;
    int *p = &i;
    long long_address = reinterpret_cast<long>(p);
    int *p2 = reinterpret_cast<int *>(long_address);

    std::cout << "原始指针：" << p << ", 转换后的整数：" << long_address << ", 转换回的指针：" << p2 << std::endl;

    return 0;
}
```

`4. const_cast`：<br>
const_cast主要用于修改类型的const属性。它允许将const修饰符从对象的类型中移除，从而可以对原本不可修改的对象进行修改。需要注意的是，通过const_cast移除const属性后修改原本为const的对象可能会导致未定义行为，所以使用const_cast时需要谨慎。<br>
```c++
#include <iostream>

void print_value(int* value) {
    std::cout << "Value: " << *value << std::endl;
}

int main() {
    const int i = 42;
    // 以下语句无法通过编译，因为 i 是 const
    // print_value(&i);

    int* non_const_i = const_cast<int*>(&i);
    *non_const_i = 84;
    print_value(non_const_i);

    return 0;
}
```