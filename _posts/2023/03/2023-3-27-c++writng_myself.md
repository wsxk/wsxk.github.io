---
layout: post
tags: [c++]
title: "c++ 手写array & vector"
date: 2023-3-27
author: wsxk
comments: true
---

- [array](#array)
- [vector](#vector)

array和 vector<br>
不知道大伙用这两个东西用了多久<br>
两个都是c++的典型数据结构了<br>
值得好好了解一下！<br>
。。。<br>
。。。<br>


PS:`更新于2024-11-7`<br>

## array<br>
```c++
#include <iostream>

template<typename T,size_t S>
class Array {
public:
	constexpr size_t Size() const{ return S; }
	T& operator[](size_t index) { return m_Data[index]; }//必须是T&,这样允许data[i]= 4的形式
	const T& operator[](size_t index)const { return m_Data[index]; }//为const变量准备的
	T* Data() { return m_Data; }
	const T* Data() const { return m_Data; }
private:
	T m_Data[S];
};

int main() {
	Array<int, 5> data;
	memset(data.Data(), 2, data.Size()*sizeof(int));
	for (int i = 0; i < data.Size(); i++) {
		std::cout << data[i] << std::endl;
	}

}
```

## vector<br>
```c++
#pragma once
#include <iostream>
template<typename T>
class Vector {
public:
	Vector() {
		ReAlloc(2);
	}
	~Vector() {
		Clear();
		//delete[] m_data;
		::operator delete(m_data, m_capacity * sizeof(T));
	}
	void push_back(const T& value) {
		if (m_size >= m_capacity) {
			ReAlloc(m_capacity + m_capacity / 2 ); //grow 1.5times
		}
		m_data[m_size] = value;
		m_size++;
	}
	void push_back( T&& value) {
		if (m_size >= m_capacity) {
			ReAlloc(m_capacity + m_capacity / 2); //grow 1.5times
		}
		m_data[m_size] = std::move(value);
		m_size++;
	}

	template<typename ... Args>
	T& EmplaceBack(Args&&... args) {
		if (m_size >= m_capacity) {
			ReAlloc(m_capacity + m_capacity / 2); //grow 1.5times
		}
		//m_data[m_size] = T(std::forward<Args>(args)...);
		new(&m_data[m_size])  T(std::forward<Args>(args)...);
		return m_data[m_size++];
	}
	void PopBack() {
		if (m_size > 0) {
			m_size--;
			m_data[m_size].~T();
		}
	}

	void Clear() {
		for (int i = 0; i < m_size; i++) {
			m_data[i].~T();
		}
		m_size = 0;
	}

	size_t Size() {
		return m_size;
	}
	T& operator[](size_t index) {
		return m_data[index];
	}
	const T& operator[](size_t index)const {
		return m_data[index];
	}
private:
	void ReAlloc(size_t NewCapacity) {
		//1. alloc new
		//2. copy
		//3. delete old
		if (NewCapacity < m_size) {
			m_size = NewCapacity;
		}
		//T* newBlock = new T[NewCapacity];
		T* newBlock = (T*)::operator new(NewCapacity * sizeof(T));
		for (int i = 0; i < m_size; i++) {//即使是缩减容量，也是可以的
			newBlock[i] = std::move(m_data[i]);//确保每个类型的构造/赋值函数可以被正确调用，因此不要使用memcpy
		}
		for (int i = 0; i < m_size; i++) {
			m_data[i].~T();
		}
		//delete[] m_data;
		::operator delete(m_data, m_capacity * sizeof(T));
		m_data = newBlock;
		m_capacity = NewCapacity;
	}
	T* m_data=nullptr;
	size_t m_size = 0;
	size_t m_capacity = 0;
};
```
实验例子:
```c++
#include "Vector.h"
#include <iostream>

struct Vector3 {
	float x , y , z;
	Vector3() { std::cout << "create" << std::endl; };
	Vector3(float value) :x(value), y(value), z(value) { std::cout << "create" << std::endl; }
	Vector3(float x, float y, float z) :x(x), y(y), z(z) { std::cout << "create" << std::endl; }
	Vector3(const Vector3& other) :x(other.x), y(other.y), z(other.z) {
		std::cout << "copy" << std::endl;
	}
	Vector3(Vector3&& other) :x(other.x), y(other.y), z(other.z) {
		std::cout << "move" << std::endl;
	}
	~Vector3() {
		std::cout << "destroy" << std::endl;
	}
	Vector3& operator=(const Vector3& other) {
		std::cout << "copy" << std::endl;
		x = other.x;
		y = other.y;
		z = other.z;
		return *this;
	}
	Vector3& operator=(Vector3&& other) {
		std::cout << "move" << std::endl;
		x = other.x;
		y = other.y;
		z = other.z;
		return *this;
	}
};
template<typename T>
void PrintVector(Vector<T>& vector) {
	for (int i = 0; i < vector.Size(); i++) {
		std::cout << vector[i] << std::endl;
	}
	std::cout << "----------------------------------------" << std::endl;
}
void PrintVector(Vector<Vector3>& vector) {
	for (int i = 0; i < vector.Size(); i++) {
		std::cout << vector[i].x << " " << vector[i].y << " " << vector[i].z << std::endl;
	}
	std::cout << "----------------------------------------" << std::endl;
}


int main() {
	//Vector<std::string> vector;
	//vector.push_back("wsxk");
	//vector.push_back("c++");
	//vector.push_back("test");
	//PrintVector(vector);
	Vector<Vector3> vector;
	//vector.push_back(Vector3(1.0f));
	//vector.push_back(Vector3());
	//vector.push_back(Vector3(2.5, 3.5, 4.5));
	vector.EmplaceBack(1.0, 2.0, 3.0);
	vector.EmplaceBack(1.0);
	vector.EmplaceBack(2.5, 3.5, 4.5);
	PrintVector(vector);
}
```

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>
