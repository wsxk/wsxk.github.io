---
layout: post
tags: [c++]
title: "c++ 机制逆向分析"
date: 2023-5-15
author: wsxk
comments: true
---

- [前言](#前言)
- [虚函数](#虚函数)
	- [1. 虚函数的机制](#1-虚函数的机制)
	- [2. 全局变量类型有虚函数](#2-全局变量类型有虚函数)
- [继承和多重继承](#继承和多重继承)
	- [1. 识别类与类之间的关系](#1-识别类与类之间的关系)
		- [有虚函数的继承关系](#有虚函数的继承关系)
	- [2.多重继承](#2多重继承)
	- [3. 抽象类](#3-抽象类)
	- [4. 虚继承](#4-虚继承)

## 前言<br>
本节对先前学习过的c++类的相关的底层实现进行学习。<br>
**具体而言，就是c++中类机制的汇编代码实现**<br>
其实目的是为了更深刻得理解c++<br>

## 虚函数<br>
在C++中，使用关键字virtual声明函数为虚函数。当类中定义有虚函数时，编译器会将该类中所有虚函数的首地址保存在一张地址表中，这张表被称为虚函数地址表，简称虚表。同时，编译器还会在类中添加一个隐藏数据成员，称为虚表指针。该指针保存着虚表的首地址，用于记录和查找虚函数。
### 1. 虚函数的机制<br>
在 `VS2022` `x64 debug`上进行实验。<br>
```c++
#include <stdio.h>
class Person {
public:
	virtual int getAge() { //虚函数定义
		return age;
	}
	virtual void setAge(int age) { //虚函数定义
		this->age = age;
	}
private:
	int age;
};
int main(int argc, char* argv[]) {
	Person person;
	return 0;
}
```
在调试时可以发现：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523134458.png)
`person`类中不仅有`age`参数，还有`__vfptr`,即虚表指针。<br>
如果把程序拖入IDA，可以发现`__vfptr`指向一个函数指针数组，里面存放了内容如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523134654.png)
所以虚表指针和虚表的关系如下所示:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523134727.png)
**1. 初始化虚表的操作在构造函数中实现，因此在用户没有编写构造函数时，因为必须初始化虚表指针，所以编译器会提供默认的构造函数，以完成虚表指针的初始化。**<br>
**2. 另外值得一提的是，虚表指针保存在了类的首部**<br>
**3. 在析构函数中，同样有给虚表重新赋值的汇编代码被发现了。构造函数赋值很好理解，当时虚表没有初始化，析构函数为什么也要重新给虚表赋值呢？**

***因为编译器无法预知这个子类以后是否会被其他类继承，如果被继承，原来的子类就成了父类，在执行析构函数时会先执行当前对象的析构函数，然后向祖父类的方向按继承线路逐层调用各类析构函数，当前对象的析构函数开始执行时，其虚表也是当前对象的，所以执行到父类的析构函数时，虚表必须改写为父类的虚表。编译器产生的类实现代码，必须能够适应将来不可预知的对象关系，故在每个对象的析构函数内，要加入自己虚表的代码。*** <br>

### 2. 全局变量类型有虚函数<br>
```c++
#include <stdio.h>
class Global {
public:
Global() { //无参构造函数
printf("Global\n");
}
Global(int n) { //有参构造函数
printf("Global(int n) %d\n", n);
}
Global(const char *s) { //有参构造函数
printf("Global(char *s) %s\n", s);
}
virtual ~Global() { //虚析构函数
printf("~Global()\n");
}
void show(){
printf("Object Addr: 0x%p", this);
}
};
Global g_global1;
Global g_global2(10);
Global g_global3("hello C++");
int main(int argc, char* argv[]) {
g_global1.show();
g_global2.show();
g_global3.show();
return 0;
}
```
它的汇编代码如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523140038.png)<br>
根据刚才的说法，虚表需要在构造函数时被初始化，那么，全局变量调用构造函数是什么时候呢？<br>
在IDA中进行分析，我们发现，构造全局变量是在一个构造代理函数中进行的:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523140315.png)
**在构造代理函数中，调用了`j_atexit`函数，这个函数用于注册类的析构代理函数，用于在main函数结束时被自动调用，用于析构全局变量类型,这种情况下，rcx的值就是对于类的析构代理函数**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523140535.png)

## 继承和多重继承<br>
在C++的继承关系中，子类具备父类所有成员数据和成员函数。子类对象可以直接使用父类中声明为 公有（public）和保护（protected）的数据成员与成员函数。对于在父类中声明为私有（private）的成员，虽然子类对象无法直接访问，但是在子类对象的内存结构中，父类私有的成员数据依然存在。C++语法规定的访问控制仅限于编译层面，在编译的过程中由编译器进行语法检查，因此访问控制不会影响对象的内存结构。<br>
### 1. 识别类与类之间的关系<br>
```c++
#include <stdio.h>
class Base { //基类定义
public:
	Base() {
		printf("Base\n");
	}
	~Base() {
		printf("~Base\n");
	}
	void setNumber(int n) {
		base = n;
	}
	int getNumber() {
		return base;
	}
public:
	int base;
private:
	int test;
};
class Derive : public Base { //派生类定义
public:
	void showNumber(int n) {
		setNumber(n);
		derive = n + 1;
		printf("%d\n", getNumber());
		printf("%d\n", derive);
	}
public:
	int derive;
};
int main(int argc, char* argv[]) {
	Derive derive;
	derive.showNumber(argc);
	return 0;
}
```
调试结果如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523141307.png)
其构造函数中，汇编代码如下图所示:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523142046.png)
在构造函数中：<br>
**构造顺序：先构造父类，然后按声明顺序构造成员对象和初始化列表中指定的成员，最后才是自身的构造代码**<br>
**析构顺序：首先调用自身的析构函数，然后调用成员对象的析构函数，最后调用父类的析构函数**<br>

#### 有虚函数的继承关系<br>
```c++
#include <stdio.h>
 // 基类——“人”类
class Person {
public:
	Person() {
		showSpeak(); //调用虚函数，不多态
	}
	virtual ~Person() {
	}
	virtual void showSpeak() {
		printf("Speak No\n");
	}
};

class Chinese : public Person { // 中国人：继承自人类
public:
	Chinese() {}
	virtual ~Chinese() {}
	virtual void showSpeak() { // 覆盖基类虚函数
		printf("Speak Chinese\r\n");
	}
};
class American : public Person { //美国人：继承自人类
public:
	American() {}
	virtual ~American() {}
	virtual void showSpeak() { //覆盖基类虚函数
		printf("Speak American\r\n");
	}
};
class German : public Person { //德国人：继承自人类
public:
	German() {}
	virtual ~German() {}
	virtual void showSpeak() { //覆盖基类虚函数
		printf("Speak German\r\n");
	}
};
void speak(Person* person) { //根据虚表信息获取虚函数首地址并调用
	person->showSpeak();
}
int main(int argc, char* argv[]) {
	Chinese chinese;
	American american;
	German german;
	speak(&chinese);
	speak(&american);
	speak(&german);
	return 0;
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230525232518.png)<br>
这种情况下，因为不同类型都有自己的虚表指针，通过索引虚表指针中得到自己应该调用的showSpeak函数，这就是虚表的作用。<br>
**值得一提的是，在子类构造函数中，会先调用父类构造函数，父类构造函数首先将子类对象中的虚表填为父类对象虚表地址，在父类构造函数中，如果存在虚函数调用，就能够成功的调用父类的函数，而不是子类函数**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230525232716.png)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523143223.png)

### 2.多重继承<br>
```c++
#include <stdio.h>
class Sofa {
public:
Sofa() {
color = 2;
}
virtual ~Sofa() { // 沙发类虚析构函数
printf("virtual ~Sofa()\n");
}
virtual int getColor() { // 获取沙发颜色
return color;
}
virtual int sitDown() { // 沙发可以坐下休息
return printf("Sit down and rest your legs\r\n");
}
protected:
int color; // 沙发类成员变量
};
//定义床类
class Bed {
public:
Bed() {
length = 4;
width = 5;
}
virtual ~Bed() { //床类虚析构函数
printf("virtual ~Bed()\n");
}
virtual int getArea() { //获取床面积
return length * width;
}
virtual int sleep() { //床可以用来睡觉
return printf("go to sleep\r\n");
}
protected:
int length; //床类成员变量
int width;
};
//子类沙发床定义，派生自Sofa类和Bed类
class SofaBed : public Sofa, public Bed{
public:
SofaBed() {
height = 6;
}
virtual ~SofaBed(){ //沙发床类的虚析构函数
printf("virtual ~SofaBed()\n");
}
virtual int sitDown() { //沙发可以坐下休息
return printf("Sit down on the sofa bed\r\n");
}
virtual int sleep() { //床可以用来睡觉
return printf("go to sleep on the sofa bed\r\n");
}
virtual int getHeight() {
return height;
}
protected:
int height;
};
int main(int argc, char* argv[]) {
SofaBed sofabed;
return 0;
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523144432.png)
调试可以发现，多重继承时候，按顺序排列每个类。<br>
通过ida逆向分析发现：<br>
根据继承关系的顺序，先调用父类Sofa的构造函数。在调用另一个父类Bed时，并不是直接将对象的首地址作为this指针传递，而是向后调整了父类Sofa的长度，以调整后的地址值作为this指针，最后再调用父类Bed的构造函数。因为有了两个父类，所以子类在继承时也将它们的虚表指针一起继承了过来，也就有了两个虚表指针。可见，在多重继承中，子类虚表指针的个数取决于继承的父类的个数，有几个父类便会出现几个虚表指针<br>
在进行多态操作时，会把指针进行前移或后移，指向那个的位置。<br>

### 3. 抽象类<br>
这涉及纯虚函数的构造:<br>
```c++
// C++ 源码
#include <stdio.h>
class AbstractBase {
public:
AbstractBase() {
printf("AbstractBase()");
}
virtual void show() = 0; //定义纯虚函数
};
class VirtualChild : public AbstractBase { //定义继承抽象类的子类
public:
virtual void show() { //实现纯虚函数
printf("抽象类分析\n");
}
};
int main(int argc, char* argv[]) {
VirtualChild obj;
obj.show();
return 0;
}
```
通过逆向发现<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523144932.png)<br>
纯虚函数其实在虚表中也有对应的函数，只不过这个函数什么都没干<br>

### 4. 虚继承<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523145058.png)
出现这种继承后就有点难办了。<br>
类D属于多重继承中的子类，其父类为类B和类C，类B和类C拥有同一个父类A。在菱形继承中，一个派生类中保留间接基类的多份同名成员，虽然可以在不同的成员变量中分别存放不同的数据，但大多数情况下这是多余的。这是因为保留多份成员变量不仅占用较多的存储空间，还容易产生命名冲突。为了解决多继承时的命名冲突和冗余数据问题，C++提出了虚继承，使得在派生类中只保留一份间接基类的成员。<br>
```c++
//C++源码
#include <stdio.h>
//定义家具类，虚基类，等同于类A
class Furniture {
public:
	Furniture() {
		printf("Furniture::Furniture()\n");
		price = 0;
	}
	virtual ~Furniture() { //家具类的虚析构函数
		printf("Furniture::~Furniture()\n");
	}
	virtual int getPrice() { //获取家具价格
		printf("Furniture::getPrice()\n");
		return price;
	};
protected:
	int price; //家具类的成员变量
};
//定义沙发类，继承自类Furniture，等同于类B
class Sofa : virtual public Furniture {
public:
	Sofa() {
		printf("Sofa::Sofa()\n");
		price = 1;
		color = 2;
	}
	virtual ~Sofa() { //沙发类虚析构函数
		printf("Sofa::~Sofa()\n");
	}
	virtual int getColor() { //获取沙发颜色
		printf("Sofa::getColor()\n");
		return color;
	}
	virtual int sitDown() { //沙发可以坐下休息
		return printf("Sofa::sitDown()\n");
	}
protected:
	int color; // 沙发类成员变量
};
//定义床类，继承自类Furniture，等同于类C
class Bed : virtual public Furniture {
public:
	Bed() {
		printf("Bed::Bed()\n");
		price = 3;
		length = 4;
		width = 5;
	}
	virtual ~Bed() { //床类的虚析构函数
		printf("Bed::~Bed()\n");
	}
	virtual int getArea() { //获取床面积
		printf("Bed::getArea()\n");
		return length * width;
	}
	virtual int sleep() { //床可以用来睡觉
		return printf("Bed::sleep()\n");
	}
protected:
	int length; //床类成员变量
	int width;
};
//子类沙发床的定义，派生自类Sofa和类Bed，等同于类D
class SofaBed : public Sofa, public Bed {
public:
	SofaBed() {
		printf("SofaBed::SofaBed()\n");
		height = 6;
	}
	virtual ~SofaBed() { //沙发床类的虚析构函数
		printf("SofaBed::~SofaBed()\n");
	}
	virtual int sitDown() { //沙发可以坐下休息
		return printf("SofaBed::sitDown()\n");
	}
	virtual int sleep() { //床可以用来睡觉
		return printf("SofaBed::sleep()\n");
	}
	virtual int getHeight() {
		printf("SofaBed::getHeight()\n");
		return height;
	}
protected:
	int height; //沙发类的成员变量
};
int main(int argc, char* argv[]) {
	SofaBed sofabed;
	Furniture* p1 = &sofabed; //转换成虚基类指针
	Sofa* p2 = &sofabed; //转换成父类指针
	Bed* p3 = &sofabed; //转换成父类指针
	printf("%p %p %p\n", p1, p2, p3);
	return 0;
}
```
这个代码中，`Sofa`类和`Bed`类都使用了虚继承的方式(`virtual`)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523145352.png)
实际上 类的结构如下所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230523150552.png)
**虚基类编译表中有两项，第一项为-4，即虚基类偏移表所属类对应的对象首地址相对于虚基类偏移表的偏移值；第二项保存的是虚基类对象首地址相对于虚基类偏移表的偏移值。**<br>
如果观察汇编代码，可以发现，编译器采取编译标志的方法，构造一次`Furniture`类就不会再其他子类构造时，构造`Furniture`了。<br>
虚继承构造时Furniture、Sofa（根据标记跳过Furniture构造）、Bed（根据标记跳过Furniture构造）、SofaBed自身<br>
虚继承结构中子类的析构函数执行流程并没有像构造函数那样使用标记防止重复析构，而是将虚基类放在最后调用。先依次执行两个父类Bed和Sofa的析构函数，然后执行虚基类的析构函数。<br>