---
layout: post
tags: [c++]
title: "c++ 函数指针 & lambda & namespace & thread & 计时 & unit test & visual unit test"
author: wsxk
comments: true
date: 2023-3-17
---

- [函数指针](#函数指针)
- [lambda函数](#lambda函数)
	- [lambda作为迭代器的判断条件](#lambda作为迭代器的判断条件)
- [namespace](#namespace)
- [thread](#thread)
- [计时](#计时)
- [unit test](#unit-test)
	- [测试new和make\_shred](#测试new和make_shred)
	- [unique\_ptr 和 shared\_ptr对比](#unique_ptr-和-shared_ptr对比)
- [visual unit test](#visual-unit-test)


## 函数指针<br>
```c++
#include <iostream>

void Hello_world() {
	std::cout << "hello world!" << std::endl;
}

int main() {
	typedef void(*HelloWorld)();
	auto func = Hello_world;
	void (*p)() = &Hello_world;
	func();
	p();
	HelloWorld c = Hello_world;
	c();
}
```

函数指针还可以这样用
```c++
#include <iostream>
#include <vector>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,void(*func)(int)){
	for (int value : values) {
		func(value);
	}
}

int main() {
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, Hello_world);
}
```

## lambda函数<br>
lambda函数其实是一种匿名函数，我们写一个简短的函数，又不会在其他地方用，就适合用它。<br>
上面的内容可以改成如下形式<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,void(*func)(int)){
	for (int value : values) {
		func(value);
	}
}

int main() {
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [](int value) {std::cout << value << std::endl; });
}
```
lambda的更详细信息可以去[https://en.cppreference.com/w/](https://en.cppreference.com/w/)看<br>
`[]`用于捕获其他变量，常用的有`=（值传入所有变量）` `&（引用传入所有变量）`<br>
`()`用于传参<br>
`{}`就是一个普通函数体<br>

```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,std::function<void(int)>func){
	for (int value : values) {
		func(value);
	}
}

int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [=](int value) {std::cout << value << std::endl; std::cout << a << std::endl; });
}
```
但是注意的是，通过值传递是无法修改变量的，在函数体内也不能做类似于`a=5`的操作<br>
添加`mutable`关键字后就可以了。<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,std::function<void(int)>func){
	for (int value : values) {
		func(value);
	}
}

int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [=](int value) mutable {a = 5; std::cout << value << std::endl; std::cout << a << std::endl; });
	std::cout << a << std::endl; //当然不改变值传递的本质，仍然为10
}
```
通过引用传递的是会修改的<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>

void Hello_world(int value) {
	std::cout << "hello world! value:" <<value<< std::endl;
}

void PrintValue(std::vector<int>&values,std::function<void(int)>func){
	for (int value : values) {
		func(value);
	}
}

int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	PrintValue(values, [&](int value) {a = 5; std::cout << value << std::endl; std::cout << a << std::endl; });
	std::cout << a << std::endl; //是5
}
```

### lambda作为迭代器的判断条件<br>
```c++
#include <iostream>
#include <vector>
#include <algorithm>
#include <functional>
int main() {
	int a = 10;
	std::vector<int> values = { 1,2,3,4,5 };
	auto it = std::find_if(values.begin(), values.end(), [](int value) {return value > 3; });
	std::cout << it[1] << std::endl;
}
```

## namespace<br>
`namespace`其实就起一个标识函数的作用<br>
```c++
#include<iostream>
#include <string>
#include <algorithm>

namespace apple {
	void print(const std::string& text) {
		std::cout << text << std::endl;
	}
}

namespace orange {
	void print(const char* text) {
		std::string temp = text;
		std::reverse(temp.begin(), temp.end());
		std::cout << temp << std::endl;
	}
}

int main() {
	apple::print("wsxk");
	orange::print("wsxk");
	using namespace apple;
	namespace a = apple;
	print("wsxk");
	a::print("wsxk");
}
```

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


## visual unit test<br>
取自:[https://gist.github.com/TheCherno/31f135eea6ee729ab5f26a6908eb3a5e](https://gist.github.com/TheCherno/31f135eea6ee729ab5f26a6908eb3a5e)<br>
代码如下:
```c++
//
// Basic instrumentation profiler by Cherno

// Usage: include this header file somewhere in your code (eg. precompiled header), and then use like:
//
// Instrumentor::Get().BeginSession("Session Name");        // Begin session 
// {
//     InstrumentationTimer timer("Profiled Scope Name");   // Place code like this in scopes you'd like to include in profiling
//     // Code
// }
// Instrumentor::Get().EndSession();                        // End Session
//
// You will probably want to macro-fy this, to switch on/off easily and use things like __FUNCSIG__ for the profile name.
//
#pragma once

#include <string>
#include <chrono>
#include <algorithm>
#include <fstream>

#include <thread>

struct ProfileResult
{
    std::string Name;
    long long Start, End;
    uint32_t ThreadID;
};

struct InstrumentationSession
{
    std::string Name;
};

class Instrumentor
{
private:
    InstrumentationSession* m_CurrentSession;
    std::ofstream m_OutputStream;
    int m_ProfileCount;
public:
    Instrumentor()
        : m_CurrentSession(nullptr), m_ProfileCount(0)
    {
    }

    void BeginSession(const std::string& name, const std::string& filepath = "results.json")
    {
        m_OutputStream.open(filepath);
        WriteHeader();
        m_CurrentSession = new InstrumentationSession{ name };
    }

    void EndSession()
    {
        WriteFooter();
        m_OutputStream.close();
        delete m_CurrentSession;
        m_CurrentSession = nullptr;
        m_ProfileCount = 0;
    }

    void WriteProfile(const ProfileResult& result)
    {
        if (m_ProfileCount++ > 0)
            m_OutputStream << ",";

        std::string name = result.Name;
        std::replace(name.begin(), name.end(), '"', '\'');

        m_OutputStream << "{";
        m_OutputStream << "\"cat\":\"function\",";
        m_OutputStream << "\"dur\":" << (result.End - result.Start) << ',';
        m_OutputStream << "\"name\":\"" << name << "\",";
        m_OutputStream << "\"ph\":\"X\",";
        m_OutputStream << "\"pid\":0,";
        m_OutputStream << "\"tid\":" << result.ThreadID << ",";
        m_OutputStream << "\"ts\":" << result.Start;
        m_OutputStream << "}";

        m_OutputStream.flush();
    }

    void WriteHeader()
    {
        m_OutputStream << "{\"otherData\": {},\"traceEvents\":[";
        m_OutputStream.flush();
    }

    void WriteFooter()
    {
        m_OutputStream << "]}";
        m_OutputStream.flush();
    }

    static Instrumentor& Get()
    {
        static Instrumentor instance;
        return instance;
    }
};

class InstrumentationTimer
{
public:
    InstrumentationTimer(const char* name)
        : m_Name(name), m_Stopped(false)
    {
        m_StartTimepoint = std::chrono::high_resolution_clock::now();
    }

    ~InstrumentationTimer()
    {
        if (!m_Stopped)
            Stop();
    }

    void Stop()
    {
        auto endTimepoint = std::chrono::high_resolution_clock::now();

        long long start = std::chrono::time_point_cast<std::chrono::microseconds>(m_StartTimepoint).time_since_epoch().count();
        long long end = std::chrono::time_point_cast<std::chrono::microseconds>(endTimepoint).time_since_epoch().count();

        uint32_t threadID = std::hash<std::thread::id>{}(std::this_thread::get_id());
        Instrumentor::Get().WriteProfile({ m_Name, start, end, threadID });

        m_Stopped = true;
    }
private:
    const char* m_Name;
    std::chrono::time_point<std::chrono::high_resolution_clock> m_StartTimepoint;
    bool m_Stopped;
};
```
生成的result.json 拖入 chrome浏览器`chrome://tracing`即可分析<br>
还有一个衍生版本:<br>
[https://github.com/GavinSun0921/InstrumentorTimer](https://github.com/GavinSun0921/InstrumentorTimer)<br>