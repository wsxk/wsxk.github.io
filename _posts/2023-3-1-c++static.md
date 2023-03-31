---
layout: post
tags: [c++]
title: "c++ static"
date: 2023-3-1
author: wsxk
comments: true
---

- [c++中的全局静态](#c中的全局静态)
- [c++类中的static](#c类中的static)
- [c++中的局部static](#c中的局部static)


## c++中的全局静态<br>
```c++
static int i;
static void func(){
    ;
};
```
用在全局变量当中表明只在当前翻译单元（即当前文件中可见<br>

## c++类中的static<br>
```c++
struct Entity {
	static int x, y;
	static void print() {
		std::cout << x << y << std::endl;
	}
};

int Entity::x;
int Entity::y;
```
**表明所有类的实例共享这一个静态变量。**<br>
使用时需要用如下格式:<br>
```c++
Entity::x = 4;
Entity::y = 5;
Entity::print();
```

**注意，类当中的静态函数不能访问非静态变量，因为他不知道非静态变量指向谁**<br>


## c++中的局部static<br>
```c++
void func(){
    static int i =0;
    i++;
    std::cout<<i<<std::endl;
}
```
**局部static只会创建一次并一直到程序终止时都存活，只不过只能在当前函数中访问它。**<br>