---
layout: post
tags: [c++]
title: "c++ 左值和右值 & argument evaluation order & 移动语义 & 移动赋值运算符"
date: 2023-3-26
author: wsxk
comments: true
---


- [左值和右值](#左值和右值)
	- [左值引用和右值引用](#左值引用和右值引用)
- [argument evaluation order](#argument-evaluation-order)
- [movement semantic](#movement-semantic)
	- [solution](#solution)
- [移动赋值运算符](#移动赋值运算符)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## 左值和右值<br>
左值其实指的是**在内存中有实际存储的值，右值则是临时值**，但其实没有严格定义<br>
另一种说法是，左值是等式左边的值，右值的等式右边的值（绝大多数情况下成立）<br>
因为左值和右值，其实出现了很多反人类的东西。<br>
```c++
#include <iostream>

int& GetValue() {
	int i = 10;
	return i;
}
void SetValue(int& value) {
}
int main() {
	int i = 10;
	int a = i;
	const int& b = 10; //同样反人类，引用只是个标签，其实等价于 int c = 10,int &b = c
	GetValue() = 6;//反人类，然而实际还是可以运作，有安全风险
	//SetValue(10);//反人类，其实这样做是不行的
}
```
**const int &中的const关键字是个很重要的东西，它允许你可以赋给他左值或者临时的右值。**<br>
```c++
#include <iostream>

void PrintName(std::string& name) {
	std::cout << name << std::endl;
}

int main() {
	std::string first_name = "wu";
	std::string second_name = "xianke";
	std::string name = first_name + second_name;
	PrintName(name);
	PrintName(first_name + second_name);//编译不通过
}
```
可以看到，first_name+second_name实际上一个右值，是一个临时变量。<br>
要想通过编译，需要在`PrintName`函数的参数前加一个关键字`const`<br>
```c++
#include <iostream>

void PrintName(const std::string& name) {
	std::cout << name << std::endl;
}

int main() {
	std::string first_name = "wu";
	std::string second_name = "xianke";
	std::string name = first_name + second_name;
	PrintName(name);
	PrintName(first_name + second_name);//编译通过
}
```

### 左值引用和右值引用<br>
`&&`表示右值引用<br>
利用c++的重载特性，编写出左值和右值都能传入的函数<br>
即使有const关键字能够兼容右值，c++还是会调用右值引用的函数<br>
```c++
#include <iostream>

void PrintName(const std::string& name) {
	std::cout << "Lvalue:" << name << std::endl;
}

void PrintName(const std::string&& name) {
	std::cout << "Rvalue:"<<name << std::endl;
}

int main() {
	std::string first_name = "wu";
	std::string second_name = "xianke";
	std::string name = first_name + second_name;
	PrintName(name);
	PrintName(first_name + second_name);
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230401164007.png)

**左值引用仅接受左值（加const可以兼容右值），右值引用只能接受右值**<br>
**值得一提的是，右值引用在传入函数内部后已经变成了左值，所以使用时需要显示的用std::move**<br>

## argument evaluation order<br>
直接说结论，永远不要写这种神秘代码!<br>
```c++
#include <iostream>

void function(int a, int b, int c,int d) {
	std::cout << "a=" << a << std::endl;
	std::cout << "b=" << b << std::endl;
	std::cout << "c=" << c << std::endl;
	std::cout << "d=" << d << std::endl;
}
int main() {
	int a = 5;
	function(a++, a--, a++,a);
}
```
这种情况下，你永远不知道传入的参数的值是多少，c++标准并没有对传参时的顺序和求值顺序进行规范！！<br>


## movement semantic<br>
```c++
#include <iostream>

class String {
private:
	char* m_data;
	size_t m_size;
public:
	String(const char* name) {
		printf("created\n");
		m_size = strlen(name);
		m_data = new char[m_size];
		memcpy(m_data, name, m_size);
	}
	String(const String& other) {
		printf("copied\n");
		m_size = other.m_size;
		m_data = new char[m_size];
		memcpy(m_data, other.m_data, m_size);
	}
	~String() {
		delete m_data;
	}
	void Print() {
		for (int i = 0; i < m_size; i++) {
			printf("%c", m_data[i]);
		}
		printf("\n");
	}
};

class Entity {
private:
	String m_string;
public:
	Entity(const String& name):m_string(name) {
	}
	void Print() {
		m_string.Print();
	}
};

int main() {
	Entity e(String("wsxk"));
	e.Print();
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230402132839.png)
可以看到，这个代码其实创建了2次`String`类型的值，这造成了浪费(实际上我们只需要分配一次内存就可以了)，我们其实只需要1个`String`类型<br>

### solution<br>
利用右值引用的move方法。<br>
```c++
#include <iostream>

class String {
private:
	char* m_data;
	size_t m_size;
public:
	String(const char* name) {
		printf("created\n");
		m_size = strlen(name);
		m_data = new char[m_size];
		memcpy(m_data, name, m_size);
	}
	String(const String& other) {
		printf("copied\n");
		m_size = other.m_size;
		m_data = new char[m_size];
		memcpy(m_data, other.m_data, m_size);
	}
	String(String&& other) noexcept {
		printf("moved\n");
		m_size = other.m_size;
		m_data = other.m_data;
		other.m_size = 0;//简单来说，就是浅拷贝
		other.m_data = nullptr;
	}
	~String() {
		printf("Destroied\n");
		delete m_data;
	}
	void Print() {
		for (int i = 0; i < m_size; i++) {
			printf("%c", m_data[i]);
		}
		printf("\n");
	}
};

class Entity {
private:
	String m_string;
public:
	Entity(const String& name):m_string(name) {
	}
	void Print() {
		m_string.Print();
	}
	Entity(String && name):m_string(name){}
};

int main() {
	Entity e(String("wsxk"));
	e.Print();
}
```
我们发现它还是一样的运行结果<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230402134550.png)

出现这个问题的根本原因是**有名字的右值引用，其实是左值,所以我们需要显示的声明它是右值**<br>
```c++
#include <iostream>

class String {
private:
	char* m_data;
	size_t m_size;
public:
	String(const char* name) {
		printf("created\n");
		m_size = strlen(name);
		m_data = new char[m_size];
		memcpy(m_data, name, m_size);
	}
	String(const String& other) {
		printf("copied\n");
		m_size = other.m_size;
		m_data = new char[m_size];
		memcpy(m_data, other.m_data, m_size);
	}
	String(String&& other) noexcept {
		printf("moved\n");
		m_size = other.m_size;
		m_data = other.m_data;
		other.m_size = 0;
		other.m_data = nullptr;
	}
	~String() {
		printf("Destroied\n");
		delete m_data;
	}
	void Print() {
		for (int i = 0; i < m_size; i++) {
			printf("%c", m_data[i]);
		}
		printf("\n");
	}
};

class Entity {
private:
	String m_string;
public:
	Entity( const String& name):m_string(name) {
	}
	void Print() {
		m_string.Print();
	}
	Entity(String && name):m_string((String &&)name){}//显示声明是右值
};

int main() {
	Entity e(String("wsxk"));
	e.Print();
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230402134834.png)

更常用的方法是<br>
```c++
#include <iostream>

class String {
private:
	char* m_data;
	size_t m_size;
public:
	String(const char* name) {
		printf("created\n");
		m_size = strlen(name);
		m_data = new char[m_size];
		memcpy(m_data, name, m_size);
	}
	String(const String& other) {
		printf("copied\n");
		m_size = other.m_size;
		m_data = new char[m_size];
		memcpy(m_data, other.m_data, m_size);
	}
	String(String&& other) noexcept {
		printf("moved\n");
		m_size = other.m_size;
		m_data = other.m_data;
		other.m_size = 0;
		other.m_data = nullptr;
	}
	~String() {
		printf("Destroied\n");
		delete m_data;
	}
	void Print() {
		for (int i = 0; i < m_size; i++) {
			printf("%c", m_data[i]);
		}
		printf("\n");
	}
};

class Entity {
private:
	String m_string;
public:
	Entity( const String& name):m_string(name) {
	}
	void Print() {
		m_string.Print();
	}
	Entity(String && name):m_string(std::move(name)){}//用std::move来包裹
};

int main() {
	Entity e(String("wsxk"));//这里String("wsxk")其实也是临时变量，是右值，所以会先调用右值引用的函数
	e.Print();
}
```

## 移动赋值运算符<br>
还是用上文用到的代码来看看情况。<br>
```c++
#include <iostream>

class String {
private:
	char* m_data;
	size_t m_size;
public:
	String(const char* name) {
		printf("created\n");
		m_size = strlen(name);
		m_data = new char[m_size];
		memcpy(m_data, name, m_size);
	}
	String(const String& other) {
		printf("copied\n");
		m_size = other.m_size;
		m_data = new char[m_size];
		memcpy(m_data, other.m_data, m_size);
	}
	String(String&& other) noexcept {
		printf("moved\n");
		m_size = other.m_size;
		m_data = other.m_data;
		other.m_size = 0;
		other.m_data = nullptr;
	}
	~String() {
		printf("Destroied\n");
		delete m_data;
	}
	void Print() {
		for (int i = 0; i < m_size; i++) {
			printf("%c", m_data[i]);
		}
		printf("\n");
	}
};

class Entity {
private:
	String m_string;
public:
	Entity(const String& name):m_string(name) {
	}
	void Print() {
		m_string.Print();
	}
	Entity(String && name):m_string(std::move(name)){}
};

int main() {
	String b = "wsxkwsxk";//create
	String s=b;// copy
	String c = std::move(b);//move
}
```
在这种情况时，我们用 `=` 其实都是创建一个新的变量，因此都是调用`String`类的构造函数。<br>
接下来还出现了类似于`c = b`的情况，其实会调用赋值运算函数。<br>
看一个具体例子:
```c++
#include <iostream>

class String {
private:
	char* m_data;
	size_t m_size;
public:
	String() = default;
	String(const char* name) {
		printf("created\n");
		m_size = strlen(name);
		m_data = new char[m_size];
		memcpy(m_data, name, m_size);
	}
	String(const String& other) {
		printf("copied\n");
		m_size = other.m_size;
		m_data = new char[m_size];
		memcpy(m_data, other.m_data, m_size);
	}
	String(String&& other) noexcept {
		printf("moved\n");
		m_size = other.m_size;
		m_data = other.m_data;
		other.m_size = 0;
		other.m_data = nullptr;
	}
	String& operator=(String&& other) {
		printf("assigned\n");
		if (this != &other) {//防止other就是这个变量，不能有a=a的情况发生，这没有意义而且使得内存遭到破坏
			delete[] m_data;//赋值前的变量可能已经分配了内存，需要提前释放
			m_size = other.m_size;
			m_data = other.m_data;
			other.m_size = 0;
			other.m_data = nullptr;
		}
		return *this;
	}
	~String() {
		printf("Destroied\n");
		delete m_data;
	}
	void Print() {
		for (int i = 0; i < m_size; i++) {
			printf("%c", m_data[i]);
		}
		printf("\n");
	}
};

class Entity {
private:
	String m_string;
public:
	Entity(const String& name):m_string(name) {
	}
	void Print() {
		m_string.Print();
	}
	Entity(String && name):m_string(std::move(name)){}
};

int main() {
	String b = "wsxkwsxk";//这个=是构造函数，创建变量时都是调用构造函数
	String c;
	c = std::move(b); // 这个=是移动赋值函数，对已存在变量进行修改都是调用 operator=
	std::cout << "c: ";
	c.Print();
	std::cout << "b: ";
	b.Print();
}
```