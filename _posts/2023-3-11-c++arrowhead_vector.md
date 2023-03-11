---
layout: post
tags: [c++]
title: "c++ ->重载&动态数组std::vector"
author: wsxk
date: 2023-3-11
comments: true
---

- [-\>重载](#-重载)
- [std::vector](#stdvector)


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
