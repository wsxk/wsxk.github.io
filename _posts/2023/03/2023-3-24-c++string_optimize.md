---
layout: post
tags: [c++]
title: "c++ std::string optimize & 单例模式 & malloc_tracing"
date: 2023-3-24
author: wsxk
comments: true
---

- [std::string optimize](#stdstring-optimize)
	- [std::view](#stdview)
	- [const char \*](#const-char-)
- [单例模式](#单例模式)
	- [随机数生成器](#随机数生成器)
- [malloc tracing](#malloc-tracing)
	- [template](#template)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## std::string optimize<br>
直接上例子<br>
```c++
#include <iostream>
#include <string>
static int mAcount = 0;
void* operator new(size_t size) {
	mAcount++;
	return malloc(size);
}
void PrintName(std::string&name){ 
	std::cout << name << std::endl;
}
int main() {
	std::string name = "wu xianke"; 
	PrintName(name);
	std::string first_name = name.substr(0, 2);
	std::string second_name = name.substr(3, 9);
	PrintName(second_name);
	std::cout << mAcount << " allocations" << std::endl;
}
```

这是一个打印自己姓名的程序，实际运行后，我们发现std::string分配了3次内存，然而，其实我们其实不需要分配那么多次内存，这样分配内存其实是很耗时的<br>
于是乎有了利用std::view化简的方法<br>
### std::view<br>
```c++
#include <iostream>
#include <string>

static int mAcount = 0;
void* operator new(size_t size) {
	mAcount++;
	return malloc(size);
}
void PrintName(std::string_view&name){ 
	std::cout << name << std::endl;
}
int main() {
	std::string name = "wu xianke"; 
	std::string_view first_name(name.c_str(),2);
	std::string_view second_name(name.c_str()+3,6);
	PrintName(second_name);
	std::cout << mAcount << " allocations" << std::endl;
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230331201941.png)

### const char *<br>
最快还得是用指针（<br>
```c++
#include <iostream>
#include <string>

static int mAcount = 0;
void* operator new(size_t size) {
	mAcount++;
	return malloc(size);
}
void PrintName(std::string_view&name){ 
	std::cout << name << std::endl;
}
int main() {
	const char * name = "wu xianke"; 
	std::string_view first_name(name,2);
	std::string_view second_name(name+3,6);
	PrintName(second_name);
	std::cout << mAcount << " allocations" << std::endl;
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230331202218.png)


## 单例模式<br>
单例模式主要使用在我们只需要一个类的实例的时候，（其实类作为模板，当然是允许有多个实例，but有的时候，我们真的只想要一个实例）<br>
**下面是一个常用的单例模式的类模板**<br>
```c++
#include <iostream>

class SingleTon {
public:
	SingleTon(const SingleTon&) = delete;//删除复制的函数，防止类似于 SingleTon a = b
	static SingleTon& Get() {//只能通过Get方法来获得唯一实例
		return SingleInstance;
	}
	void function() {
		;
	}
private:
	SingleTon() {};//构造函数放在private，所以无法构造
	float m_float = 0.0f;
	static SingleTon SingleInstance;
};
SingleTon SingleTon::SingleInstance;

int main() {
	SingleTon &a = SingleTon::Get();
	return 0;
}
```

### 随机数生成器<br>
随机数生成器是很适合单例模式的场景<br>
```c++
#include <iostream>

class Random {
public:
	Random(const Random&) = delete;//删除复制的函数，防止类似于 SingleTon a = b
	static Random& Get() {
		return SingleInstance;
	}
	static float Float() {//直接用Random::Float()
		return Get().IFloat();
	}
private:
	float IFloat() { //假装这是一个可以随机生成随机数的函数
		return m_float;
	}
	Random() {};//构造函数放在private，所以无法构造
	float m_float = 0.5f;
	static Random SingleInstance;
};
Random Random::SingleInstance;

int main() {
	float a = Random::Float();
	std::cout << a << std::endl;
	return 0;
}
```

## malloc tracing<br>
有时候我们想要追踪程序到底使用了多少内存，或者某个类型的变量使用了多少内存，尽管我们可以用其他的工具，我们也可以简单的写一段代码来进行跟踪。<br>
**代码跟踪的原理是，利用重写operator来添加信息**<br>
下面是一个例子<br>
```c++
#include <iostream>

void* operator new(size_t size) {
	std::cout << "malloc " << size << " bytes \n";
	return malloc(size);
}
void operator delete(void* memory,size_t size) {
	std::cout << "free " << size << " bytes\n";
	free(memory);
}

struct Object {
	int x, y, z;
};

int main() {
	{
		std::unique_ptr<Object*> a = std::make_unique<Object*>();
	}
	return 0;
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230401150206.png)
其实到这里，有人可能会问，为什么delete不是不加参数吗，咋后面还能再加一个`size_t size`，其实这是`c++17`的新特性<br>
**在C++17中，operator delete可以包含一个额外的 size 参数。这个参数表示要释放的对象的大小。这里是为了优化内存分配器的性能。如果您为类定义了一个带有 size 参数的 operator delete，那么C++17及更新版本的编译器会优先选择这个版本。**<br>

### template<br>
有一个模板可供使用<br>
```c++
#include <iostream>

struct MemoryTracing {
	size_t TotalAlloc = 0;
	size_t TotalFree = 0;
	void CurrentUsage() {
		std::cout << "currently use " << TotalAlloc - TotalFree << " bytes\n";
	}
};

static MemoryTracing AllocaionMetrics;

void* operator new(size_t size) {
	AllocaionMetrics.TotalAlloc += size;
	return malloc(size);
}
void operator delete(void* memory,size_t size) {
	AllocaionMetrics.TotalFree += size;
	free(memory);
}

void PrintMemoryUsage() {
	AllocaionMetrics.CurrentUsage();
}

struct Object {
	int x, y, z;
};

int main() {
	PrintMemoryUsage();
	{
		std::unique_ptr<Object*> a = std::make_unique<Object*>();
		PrintMemoryUsage();
	}
	PrintMemoryUsage();
	return 0;
}
```