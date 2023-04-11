---
layout: post
tags: [c++]
title: "c++ 智能指针 & 拷贝构造函数"
date: 2023-3-10
author: wsxk
comments: true
---

- [智能指针](#智能指针)
	- [unique\_ptr](#unique_ptr)
	- [shared\_ptr](#shared_ptr)
	- [weak\_ptr](#weak_ptr)
- [拷贝构造函数](#拷贝构造函数)



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