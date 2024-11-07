---
layout: post
tags: [c++]
title: "c++ 智能指针 & 拷贝构造函数 & ->重载 & 动态数组std::vector & template & std::array"
date: 2023-3-10
author: wsxk
comments: true
---

- [智能指针](#智能指针)
	- [unique\_ptr](#unique_ptr)
	- [shared\_ptr](#shared_ptr)
	- [weak\_ptr](#weak_ptr)
- [拷贝构造函数](#拷贝构造函数)
- [-\>重载](#-重载)
- [std::vector](#stdvector)
- [std::vector优化](#stdvector优化)
- [template](#template)
- [std::array](#stdarray)
- [array 和 vector的区别](#array-和-vector的区别)

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## 智能指针<br>
c++中，我们可以使用`new` `delete`关键字来从heap上分配内存。<br>
然而，有时候我们会忘记`delete`，所以有了智能指针。<br>
智能指针有三类**unique_ptr,shared_ptr,weak_ptr**<br>

### unique_ptr<br>
在当前作用域创建的`unique_ptr`，在作用域结束时会自动销毁分配内存。<br>
### shared_ptr<br>
所有指向同一个对象的 shared_ptr指针，只有在所有的 `shared_ptr`离开作用域后，才会自动销毁<br>
### weak_ptr<br>
同`shared_ptr`，但是不会增加计数器的技术，当所有`shared_ptr`离开作用域后，即使`weak_pointer`仍然存在，也会自动销毁指向对象。<br>
```c++
#include <iostream>

class Entity {
public:
	Entity() {
		std::cout << "create Entity!" << std::endl;
	}
	~Entity() {
		std::cout << "destroy Entity!" << std::endl;
	}
	void print() {

	}
};
int main() {
    {
        std::shared_ptr<Entity> shared_pointer;
        {
            std::shared_ptr<Entity> e0  = std::make_shared<Entity>();
			shared_pointer = e0;
        }
    }
	{
		//std::shared_ptr<Entity> shared_pointer(new Entity());//导致分配2次，影响性能
		std::weak_ptr<Entity> weak_pointer;
		{
			std::shared_ptr<Entity> e0  = std::make_shared<Entity>();
			weak_pointer = e0;
		}
	}
	{
		//std::unique_ptr<Entity> entity(new Entity); //可能有异常安全问题
		std::unique_ptr<Entity> entity = std::make_unique<Entity>();
	}
}
```


## 拷贝构造函数<br>
拷贝构造函数涉及复制时发生`浅拷贝(拷贝指针对象时只拷贝指针)`和`深拷贝(拷贝指针对象时，不仅拷贝指针，还额外拷贝了指针所指向对象)`的问题。<br>
```c++
#include <iostream>
#include <string>

class String {
private:
	size_t m_size;
	char* m_buffer;
public:
	String(const char* string) {
		m_size = strlen(string);
		m_buffer = new char[m_size + 1];
		memcpy(m_buffer, string, m_size);
		m_buffer[m_size] = '\x00';
	}
	String(const String& other) :m_size(other.m_size) { //拷贝构造函数，深拷贝
		m_buffer = new char[m_size + 1];
		memcpy(m_buffer, other.m_buffer, m_size + 1);
	}
	char& operator[](const size_t index) {
		return m_buffer[index];
	}
	friend std::ostream& operator<<(std::ostream & stream, const String& string);
	~String() {
		delete[] m_buffer;
	}
};

std::ostream& operator<<(std::ostream & stream,const String& string) {
	stream << string.m_buffer;
	return stream;
}

int main() {
	String e("wsxk");
	String e1(e);
	e[2] = 's';
	std::cout << e << std::endl;
	std::cout << e1 << std::endl;
}
```

## ->重载<br>
```c++
#include <iostream>
#include <string>

class Entity {
private:
	int x;
public:
	void print() {
		std::cout << "hello" << std::endl;
	}
};

class ScopePtr {
private:
	Entity* m_entity;
public:
	ScopePtr(Entity* m_obj) : m_entity(m_obj) {
	}
	~ScopePtr() {
		delete m_entity;
	}
	Entity* operator->() {
		return m_entity;
	}
};
int main() {
	ScopePtr entity = new Entity();//隐式转换
	entity->print();//箭头操作符重载
}
```

## std::vector<br>
```c++
#include <iostream>
#include <vector>

class Vertex {
public:
	float x, y, z;

};
std::ostream& operator<<(std::ostream& stream, const Vertex& vertex) {
	stream << vertex.x << "," << vertex.y << "," << vertex.z;
	return stream;
}

int main() {
	std::vector<Vertex> vertices;
	vertices.push_back({ 1,2,3 });
	vertices.push_back({ 4, 5, 6 });
	for (int i = 0; i < vertices.size(); i++) {
		std::cout << vertices[i] << std::endl;
	}
	vertices.erase(vertices.begin() + 1);
	for (int i = 0; i < vertices.size(); i++) {
		std::cout << vertices[i] << std::endl;
	}
}
```

## std::vector优化<br>
```c++
#include <iostream>
#include <vector>

class Vertex {
public:
	float x, y, z;
	Vertex(const Vertex& vertex) : x(vertex.x), y(vertex.y), z(vertex.z) {
		std::cout << "vertex copyed!" << std::endl;
	}
	Vertex(float a,float b,float c) : x(a), y(b), z(c) {
	}
};
std::ostream& operator<<(std::ostream& stream, const Vertex& vertex) {
	stream << vertex.x << "," << vertex.y << "," << vertex.z;
	return stream;
}

int main() {
	std::vector<Vertex> vertices;
	vertices.push_back({ 1,2,3 });
	vertices.push_back({ 4, 5, 6 });
	vertices.push_back({ 7,8,9 });
}
```
实际上会copy 6次。<br>
预设容量后，减少一半<br>
```c++
#include <iostream>
#include <vector>

class Vertex {
public:
	float x, y, z;
	Vertex(const Vertex& vertex) : x(vertex.x), y(vertex.y), z(vertex.z) {
		std::cout << "vertex copyed!" << std::endl;
	}
	Vertex(float a,float b,float c) : x(a), y(b), z(c) {
	}
};
std::ostream& operator<<(std::ostream& stream, const Vertex& vertex) {
	stream << vertex.x << "," << vertex.y << "," << vertex.z;
	return stream;
}

int main() {
	std::vector<Vertex> vertices;
	vertices.reserve(3);
	vertices.push_back({ 1,2,3 });
	vertices.push_back({ 4, 5, 6 });
	vertices.push_back({ 7,8,9 });
}
```

通过不在栈上创建`Vertex`，直接传参进入，就不用发生拷贝<br>
```c++
#include <iostream>
#include <vector>

class Vertex {
public:
	float x, y, z;
	Vertex(const Vertex& vertex) : x(vertex.x), y(vertex.y), z(vertex.z) {
		std::cout << "vertex copyed!" << std::endl;
	}
	Vertex(float a,float b,float c) : x(a), y(b), z(c) {
	}
};
std::ostream& operator<<(std::ostream& stream, const Vertex& vertex) {
	stream << vertex.x << "," << vertex.y << "," << vertex.z;
	return stream;
}

int main() {
	std::vector<Vertex> vertices;
	vertices.reserve(3);
	vertices.emplace_back(1, 2, 3);
	vertices.emplace_back(4, 5, 6);
	vertices.emplace_back(7, 8, 9);
}
```

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