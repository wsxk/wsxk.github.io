---
layout: post
tags: [knowledge]
title: "make learing"
date: 2022-8-20
author: wsxk
comments: true
---

在大学生活中，我们平时编写程序时，是如何编译代码的呢？<br>
在linux下，往往一行命令就能解决问题: gcc -o test test.c<br>
但是如果在以后，你想要编译一个几百万行代码组成的程序，这时候要怎么办呢？光靠自己手打gcc的命令，估计一天结束你还没有编译完代码。<br>
GNU make可以帮助你解决这个问题。GNU make允许你在一个文件（makefile）上一个大型程序的编译规则，只需要一个命令（make）就能开始自动化编译流程，极大的减少了程序员编译时的工作量。由此可见make还是很重要的！！！<br>

- [make基本认识<br>](#make基本认识)
- [make常用命令选项<br>](#make常用命令选项)
- [make 规则<br>](#make-规则)
  - [1.显式规则<br>](#1显式规则)
  - [2.变量<br>](#2变量)
  - [3.隐式规则<br>](#3隐式规则)
  - [4.引用其他makefile<br>](#4引用其他makefile)
  - [5.忽略错误<br>](#5忽略错误)
  - [6.系统常量/系统变量<br>](#6系统常量系统变量)
  - [7.伪目标<br>](#7伪目标)
  - [8.模式匹配<br>](#8模式匹配)
  - [9. =与:=的区别<br>](#9-与的区别)
  - [10. make中使用shell<br>](#10-make中使用shell)
  - [11. make中的条件判断<br>](#11-make中的条件判断)
  - [11. make中的循环<br>](#11-make中的循环)
  - [12. make自定义函数<br>](#12-make自定义函数)
  - [13. make install<br>](#13-make-install)

## make基本认识<br>
使用make编译程序时，有关程序的编译信息都必须写在一个“配置”文件中，一般情况下是“配置”文件名称是 **makefile** 或 **Makefile** ，当你写好配置文件后，就可以直接使用 **make** 命令一键编译。<br>
当然了，“配置”文件名称也不一定要是上面提到的两个，比如你的程序编译的“配置”文件名称是 **test_make** ，这时候如果你要用make命令来编译时，你需要使用 **make -f test_make** 来 **显示** 地表明你的编译配置文件是哪一个，不如直接make来的方便。<br>

一个makefile文件原则上只能编译一个目标文件（有迂回的方式编译多个目标文件）。

现在举个例子，你现在有一个小项目需要编译，编译出的可执行程序叫做target，target由3个文件module1.o,module2.o,module3.o链接形成，而这3个文件又由不同的文件编译汇编组成，形成了一种**依赖树**的关系，如下图所示：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/make_structure1.jpg)

可以看到，目标文件只有target一个，但是组成target的3个模块也需要编译，所以更具体的说，**一个makefile文件原则上只能编译一个目标文件及其依赖文件** ,再换句话说，make一次只能编译一棵依赖树。<br>
这个例子的makefile文件内容如下所示：
```makefile
target : module1.o module2.o module3.o
    cc -o target module1.o module2.o module3.o

mudule1.o : module1.c service.c header1.h
    cc -c module1.c service.c

module2.o : module2.c header1.h header2.h
    cc -c module2.c

module3.o : module3.c def.h header2.h
    cc -c module3.c

.PHONY clean
clean :
    -rm target module1.o module2.o module3.o

```
注意，makefile中默认的目标文件需要写在最前面<br>

## make常用命令选项<br>
make作为一个命令，自然有很多选项可供使用。<br>
> 1. -f filename 可以指定某个特定的文件作为make的输入文件
> 2. -C dir 指定makefile所在目录
> 3. -w 显示执行前后的路径
> 4. -v make的版本号
> 5. -s 只执行命令，不会输出命令
> 6. -n 只输出命令，不执行（用作测试

## make 规则<br>
### 1.显式规则<br>
基本的编译格式如下:
```make
target : prerequisites
    command
```
target是目标文件<br>
prerequisites是target的依赖文件（比如目标可执行二进制文件test，它的依赖文件是test.c）<br>
command 是执行的命令<br>

### 2.变量<br>
就上面给出的例子而言，如果target还额外需要一个module4.o的模块文件的话，应该怎么办？大部分人肯定是想着 prerequisites和command以及清除（clean）对应的位置再额外添加一个module4.o就可以了，那么你一共需要添加3个位置。在轻量的项目中无所谓，如果在超大型项目中，一不小心少添加了个位置，你可能就要出问题了，为了方便起见，make引入了变量这个概念<br>
```makefile
objects = module1.o module2.o module3.o module4.o
```
那么原本的makefile可以变成下面的样子：
```makefile
objects = module1.o module2.o module3.o module4.o
target : $(objects)
    cc -o target $(objects)

mudule1.o : module1.c service.c header1.h
    cc -c module1.c service.c

module2.o : module2.c header1.h header2.h
    cc -c module2.c

module3.o : module3.c def.h header2.h
    cc -c module3.c

module4.o : module4.c
    cc -c module4.c

.PHONY clean
clean :
    -rm target $(objects)
```

### 3.隐式规则<br>
make是一个功能强大的编译工具，它甚至能够在一定程度上自动推导程序的编译条件，隐式规则也是因此而来<br>
举个例子，比如显示规则:
```makefile
module2.o : module2.c header1.h header2.h
    cc -c module2.c
```
其实make在得知需要编译module2.o时，必然会去寻找同名的module2.c文件（自动），且能自动推导编译命令 cc -c module2.c<br>
即make会自动的吧.o文件相应的.c文件加入编译流程中。<br>
因此，在使用隐式规则后，可以简化makefile的编写过程。<br>
那么原本的makefile可以变成下面的样子：
```makefile
objects = module1.o module2.o module3.o module4.o
target : $(objects)
    cc -o target $(objects)

mudule1.o : service.c header1.h
module2.o : header1.h header2.h
module3.o : def.h header2.h
module4.o : 

.PHONY clean
clean :
    -rm target $(objects)
```

### 4.引用其他makefile<br>
一个makefile文件可以引用其它的makefile文件。<br>
```makefile
include <filename> #filename可以带有目录和通配符
```
使用 
```makefile
bar = 4.makefile 5.makefile
include 1.makefile 2.makefile 3.makefile $(bar)
```
的形式即可<br>
要注意的是，通过include引用进入的makefile文件本质上是将其他makefile文件的内容原样替换在当前文件上下文中。

### 5.忽略错误<br>
make在执行命令中有可能会出现错误，比如：
```makefile
bar = 4.makefile 5.makefile
include 1.makefile 2.makefile 3.makefile $(bar)
```
此时，如果1.makefile的内容已经被复制黏贴到2.makefile中了，1.makefile已经被删除了,那么make在执行到这里时，如果发现1.makefile找不到，那么会报错并终止编译流程，但是理论上来说，还是能够正常编译的，因为1.makefile的内容已经存在于2.makefile中了。<br>
为了解决这个问题，有了如下规则<br>
```makefile
bar = 4.makefile 5.makefile
-include 1.makefile 2.makefile 3.makefile $(bar)
```
make中，在命令前添加 **-** 符号代表忽略报错继续执行接下来的流程。因此，即使缺少1.makefile，make也会继续include其他文件。<br>

### 6.系统常量/系统变量<br>
为了能够让你编写的makefile能够拥有跨平台的高贵能力，make中内置了许多有用的系统常量/变量来给你使用（具体体现为，不同平台的系统常量和变量的默认值不同），能够让你免除在不同平台编译一个东西而要修改makefile的情况。<br>
可以使用 `make -p` 命令来获得make内置的系统变量和系统常量的值。<br>
系统常量/系统变量 使用起来和正常的变量一样 都是 `$(variable)`
不过就个人使用而言，我感觉平时最常用的系统常量有如下几个:<br>
> 1. AS 汇编程序的名称，默认情况下为as 
> 2. CC c编译器的名称，默认情况下为 cc
> 3. CPP c预编译器的名称 默认情况下为 cc -E
> 4. CXX c++编译器名称 默认情况下为 g++
> 5. RM 文件删除程序别名 默认情况下为 rm -rf

经常使用的系统变量有如下情况:
> 1. $^  所有不重复的依赖文件，以空格分隔。
> 2. $@ 目标文件的完整名称

比如你可以使用
``` makefile
target: prerequisites
    $(CC) $^ -o $@
```
来替换原先要使用的target和prerequisites

### 7.伪目标<br>
伪目标，顾名思义，假的生成目标。<br>
伪目标最常用的地方是在使用 `make clean` 当中，该命令一般用于清除生成的目标文件，它不生成任何东西。<br>
对于伪目标的显示的声明为 `.PHONY:clean`<br>
平时，即使不声明是伪目标也是没问题的。but，当前目录下如果存在名为clean的文件的话，`make clean` 就不会如预期行动，因为make 扫描了clean文件发现该文件没有发送变化，即是最新状态，就不会执行命令。<br>
使用方式如下：
```makefile
.PHONY: clean
clean:
    rm -rf *.o
```

### 8.模式匹配<br>
模式匹配是个非常强大的功能，对于简化makefile的编写有重大意义。<br>
简单的来说，模式匹配可以写出下面的规则：
```makefile
obj = add.o sub.o multyply.o
target = calc
$(target):$(obj):
    $(CC) $^ -o $@
%.o:%.c
    $(CC) $^ -o $@
```
这里的`%.o` 和`%.cpp` 就是匹配的一种形式，匹配了所有以.o结尾的目标，以及其对应的依赖.c<br>
还可以搭配make自带的函数来实现自动化编译。<br>
```makefile
obj = $(wildcard ./*.c) #wildcard命令可以获得当前目录下所有的c文件 
obj = $(patsubst %.c,%.o,$(wildcard ./*.c)) #该命令获得当前目录下的所有c文件后，同一替换成.o文件，自动化编译，十分方便
target = calc
$(target):$(obj):
    $(CC) $^ -o $@
%.o:%.c
    $(CC) $^ -o $@
```

还有一种神奇用法:
```makefile
SRC = $(SRC:%.c=%.o)  #讲SRC中所有的.c结尾的文件替换成.o结尾的文件名，在编译时十分方便。
```

### 9. =与:=的区别<br>
讲这个之前，需要了解make时执行makefile的流程。<br>
make在扫描makefile里的信息时，首先会扫描所有的变量（并展开）再执行命令。<br>
= 指的是最终的值 跟整个makefile中的最终的变量定义有关。<br>
:= 指的是 最终的值只跟这一行之前的变量定义有关。<br>

可以看例子理解一下：
```makefile
x = foo
y = $(x) bar
x = xyz
# 输出 xyz bar
```
```makefile
x := foo
y := $(x) bar
x := xyz
#输出 foo bar
```
### 10. make中使用shell<br>
可以使用`abc = $(shell ls)` 的形式来获得linux命令的输出。<br>

### 11. make中的条件判断<br>
> 1. ifeq 是否相等
> 2. ifneq 是否不相等
> 3. ifdef 是否已定义
> 4. ifndef 是否未被定义

可以看看样例<br>
```makefile
A = 123
ifeq ($(A),123)
    echo yes
else
    echo no
endif
```
make中没有elseif类似的语法。<br>
### 11. make中的循环<br>
直接上例子
```makefile
TARGET = 1 2 3
touch $(foreach v,$(TARGET),$v_txt)
```

### 12. make自定义函数<br>
在make中，自定义函数本质上是多行命令。
```makefile
#注意 自定义函数没有返回值 但是可以把输出作为返回的内容
define func
    echo test
    echo $(1) $(2) $(3)# 1,2,3指代传入的第1、2、3个参数，0指的是本身的名字
endef

all:
    $(call func,1,2,3)
```

### 13. make install<br>
make install 的语法可以利用上面的语句才形成.
```makefile
install:$(TARGET)
    insmod $(TARGET)
```