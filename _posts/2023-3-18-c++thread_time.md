---
layout: post
tags: [c++]
title: "c++ thread & 计时 & unit test"
author: wsxk
date: 2023-3-19
comments: true
---

- [thread](#thread)
- [计时](#计时)
- [unit test](#unit-test)
	- [测试new和make\_shred](#测试new和make_shred)
	- [unique\_ptr 和 shared\_ptr对比](#unique_ptr-和-shared_ptr对比)


## thread<br>
`thread`其实主要是用来并发的执行某个事情时的工具<br>
```c++
#include <iostream>
#include <thread>

bool is_finished = false;

void Dowork() {
	std::cout << "thread id=" << std::this_thread::get_id() << std::endl;// 获得线程id
	while (!is_finished) {
		std::cout << "working..." << std::endl;
		std::this_thread::sleep_for(std::chrono::seconds(1));//睡眠1s
	}
}

int main() {
	std::thread worker(Dowork);//创建线程，传入执行函数，创建后就开始工作
	std::cin.get();
	is_finished = true;
	worker.join();// wait until the thread ends

	std::cout << "thread id=" << std::this_thread::get_id() << std::endl;

	return 0;
}
```

## 计时<br>
计时是一种非常有效的测试手段，可以帮助我们直观看到算法的耗时，以及测试用时等等<br>
这是一种平台无关的测试方法.<br>
```c++
#include <iostream>
#include <chrono>
#include <thread>

int main() {
	auto start = std::chrono::high_resolution_clock::now();
	std::this_thread::sleep_for(std::chrono::seconds(1));
	auto end = std::chrono::high_resolution_clock::now();

	std::chrono::duration<float> duration = end - start;
	std::cout << duration.count() << "s" << std::endl;
}
```
有个更简单的方法，利用**构造/析构函数自动计时**<br>
```c++
#include <iostream>
#include <chrono>
#include <thread>

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

void Function() {
	Timer time;
	for (int i = 0; i < 100; i++) {
		//std::cout << "hello world\n";
		std::cout << "hello world" << std::endl;
	}
}

int main() {
	Function();
}
```

这里有一个有趣的点，其实`std::endl`是很费时的<br>
单单改了一个符号，就能让程序执行速度快很多！<br>


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