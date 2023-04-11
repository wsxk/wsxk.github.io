---
layout: post
tags: [c++]
title: "c++ ->重载 & 动态数组std::vector"
author: wsxk
date: 2023-3-11
comments: true
---

- [-\>重载](#-重载)
- [std::vector](#stdvector)
- [std::vector优化](#stdvector优化)


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