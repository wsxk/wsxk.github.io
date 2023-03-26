---
layout: post
tags: [c++]
title: "c++ unit_test & structed bindings & optional"
author: wsxk
date: 2023-3-23
comments: true
---

- [unit test](#unit-test)
	- [测试new和make\_shred](#测试new和make_shred)
	- [unique\_ptr 和 shared\_ptr对比](#unique_ptr-和-shared_ptr对比)
- [structed bindings](#structed-bindings)
- [optional](#optional)


## unit test<br>
单元测试是一个很重要的部分，通常用来测试这个代码优化的怎么样，运行速度如何，和其他方法的速度比较等等。<br>
原理和计时一样。<br>
**值得一提的是，一般都在release模式下进行测试，debug模式添加了很多安全措施**<br>
```c++
#include <iostream>
#include <chrono>
class Timer {
public:
	Timer() {
		m_start = std::chrono::high_resolution_clock::now();
	}
	~Timer() {
		stop();
	}
	void stop() {
		m_end = std::chrono::high_resolution_clock::now();
		auto start = std::chrono::time_point_cast<std::chrono::microseconds>(m_start).time_since_epoch().count();
		auto end = std::chrono::time_point_cast<std::chrono::microseconds>(m_end).time_since_epoch().count();
		auto duration = end - start;
		double ms = duration * 0.001;
		std::cout << ms << "ms" << std::endl;
	}
private:
	std::chrono::time_point<std::chrono::high_resolution_clock> m_start;
	std::chrono::time_point<std::chrono::high_resolution_clock>	m_end;
};
int main() {
	{
		Timer TIME;
		int value = 0;
		for (int i = 0; i < 1000000; i++) {
			value += 2;
		}
	}
	//std::cout << value << std::endl;
}
```

### 测试new和make_shred<br>
```c++
#include <iostream>
#include <chrono>
#include <array>
class Timer {
public:
	Timer() {
		m_start = std::chrono::high_resolution_clock::now();
	}
	~Timer() {
		stop();
	}
	void stop() {
		m_end = std::chrono::high_resolution_clock::now();
		auto start = std::chrono::time_point_cast<std::chrono::microseconds>(m_start).time_since_epoch().count();
		auto end = std::chrono::time_point_cast<std::chrono::microseconds>(m_end).time_since_epoch().count();
		auto duration = end - start;
		double ms = duration * 0.001;
		std::cout << ms << "ms" << std::endl;
	}
private:
	std::chrono::time_point<std::chrono::high_resolution_clock> m_start;
	std::chrono::time_point<std::chrono::high_resolution_clock>	m_end;
};
int main() {
	struct Vector2 {
		float x, y;
	};
	{
		std::array<std::shared_ptr<Vector2>, 1000> ShredPtr;
		Timer time;
		for (int i = 0; i < 1000; i++) {
			ShredPtr[i] = std::make_shared<Vector2>();
		}
	}
	{
		std::array<std::shared_ptr<Vector2>, 1000> ShredPtr;
		Timer time;
		for (int i = 0; i < 1000; i++) {
			ShredPtr[i] = std::shared_ptr<Vector2>(new Vector2());
		}
	}
	//std::cout << value << std::endl;
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230326131454.png)
**事实表明，make_shared速度比new快一些**<br>

### unique_ptr 和 shared_ptr对比<br>
```c++
#include <iostream>
#include <chrono>
#include <array>
class Timer {
public:
	Timer() {
		m_start = std::chrono::high_resolution_clock::now();
	}
	~Timer() {
		stop();
	}
	void stop() {
		m_end = std::chrono::high_resolution_clock::now();
		auto start = std::chrono::time_point_cast<std::chrono::microseconds>(m_start).time_since_epoch().count();
		auto end = std::chrono::time_point_cast<std::chrono::microseconds>(m_end).time_since_epoch().count();
		auto duration = end - start;
		double ms = duration * 0.001;
		std::cout << ms << "ms" << std::endl;
	}
private:
	std::chrono::time_point<std::chrono::high_resolution_clock> m_start;
	std::chrono::time_point<std::chrono::high_resolution_clock>	m_end;
};
int main() {
	struct Vector2 {
		float x, y;
	};
	{
		std::array<std::shared_ptr<Vector2>, 1000> ShredPtr;
		Timer time;
		for (int i = 0; i < 1000; i++) {
			ShredPtr[i] = std::make_shared<Vector2>();
		}
	}
	{
		std::array<std::unique_ptr<Vector2>, 1000> ShredPtr;
		Timer time;
		for (int i = 0; i < 1000; i++) {
			ShredPtr[i] = std::make_unique<Vector2>();
		}
	}
	
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230326131725.png)
**unique_ptr要快一些**<br>

## structed bindings<br>
**structed bingdings是c++17的新特性**<br>
```c++
#include <iostream>
#include <string>
#include <tuple>

std::tuple<std::string,int> CreatePerson() {
	return { "wsxk",22 };
}

int main() {
	std::tuple<std::string, int> person = CreatePerson();
	std::string name = std::get<0>(person);
	std::cout << name << std::endl;
	int age = std::get<1>(person);
	std::cout << age << std::endl;

	std::tie(name, age) = CreatePerson();
	
	auto [name2, age2] = CreatePerson(); //structed bindings
}
```

## optional<br>
**optional同样是c++17的新特性**<br>
```c++
#include <iostream>
#include<optional>
#include <fstream>
#include <string>

std::optional<std::string> ReadFileAsString(const std::string &filepath) {
	std::ifstream stream(filepath);
	if (stream) {
		std::string result;
		// read file
		stream.close();
		return result;
	}
	return {};
}

int main() {
	std::optional<std::string> data = ReadFileAsString("data.txt");
	std::string value = data.value_or("wsxk");
	std::cout << value << std::endl;
	if (data.has_value()) {
		std::cout << "read successfully\n";
	}
	else {
		std::cout << "read failed\n";
	}

}
```