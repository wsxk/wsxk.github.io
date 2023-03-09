---
layout: post
tags: [c++]
title: "c++运算符重载"
date: 2023-3-9
author: wsxk
comments: true
---


- [operator](#operator)
- [this关键字](#this关键字)


## operator<br>
c++允许我们重新定义类中的一些运算符，例如`+,-,*,/,==,!=`的行为,从而使得编写代码时变得简洁<br>
```c++
#include <iostream>

class Vector {
public:
	float x, y;
	Vector(float a, float b):x(a),y(b) {}

	Vector Add(const Vector& other) const {
		return Vector(x + other.x, y + other.y);
	}
	Vector operator+(const Vector& ohter)const {
		return Add(ohter);
	}
	Vector Multiply(const Vector& other) const {
		return Vector(x * other.x, y * other.y);
	}
	Vector operator*(const Vector& other)const {
		return Multiply(other);
	}
	bool equal(const Vector& other)const {
		return x == other.x && y == other.y;
	}
	bool operator==(const Vector& other) const {
		return equal(other);
	}
};

int main() {
	Vector position(1.0f, 2.0f);
	Vector speed(1.1f, 1.1f);
	Vector powerup(1.2f, 1.2f);
	Vector result = position+speed*powerup;
	std::cout << result.x << " " << result.y << std::endl;
	if (position == speed) {
		std::cout << "attack" << std::endl;
	}
	else {
		std::cout << "miss" << std::endl;
	}
}
```
**值得注意的是，不应该重载过多的operator**<br>


## this关键字<br>
this关键字是类内部用于指向自身的指针<br>
```c++
#include <iostream>

class Entity {
public:
	int x, y;
	Entity(int x, int y) {
		this->x = x;
		this->y = y;
	}
};

int main() {
	Entity a(3, 4);
	std::cout << a.x << " " << a.y << std::endl;
}
```
可以方便的解决参数和变量重名的问题<br>