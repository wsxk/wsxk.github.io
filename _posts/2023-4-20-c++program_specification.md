---
layout: post
tags: [c++]
title: "c++ program specification"
author: wsxk
date: 2023-4-19
comments: true
---


- [命名风格](#命名风格)
  - [使用统一的命名风格](#使用统一的命名风格)
  - [一些约定俗成的规范](#一些约定俗成的规范)
- [文件命名](#文件命名)
- [注释](#注释)
  - [注释格式](#注释格式)
- [格式](#格式)
  - [函数大括号统一风格](#函数大括号统一风格)
  - [行宽上限不超过120](#行宽上限不超过120)


## 命名风格<br>
命名风格有很多种，常见的有大驼峰，小驼峰，蛇形<br>
**大驼峰：每个单词首字母大写，其余小写，例如 FunTest**<br>
**小驼峰：第一个单词首字母小写，其余单词首字母大写，例如 funTest**<br>
**蛇形： 所有单词字母都小写，但是单词间有下划线，例如 fun_test**<br>

### 使用统一的命名风格<br>
一般建议，***类的名称、函数名称使用大驼峰；函数参数、局部变量、类成员变量使用小驼峰；全局变量g_前缀+小驼峰***<br>

### 一些约定俗成的规范<br>
> 1. 命名时，应该使用英文单词，不是中文拼音。
> 2. 应该使用行业内存在的术语缩写，不要自创缩写。

## 文件命名<br>
源文件一般以`.cpp`结尾，头文件一般以`.h`结尾<br>
文件命名时，一般与文件中存在的类的名称一致。<br>
比如： **文件中存在 TestSpeed类，那么文件名称可以是：TestSpeed.h TestSpeed.cpp；也可以是 test_speed.h test_speed.cpp**<br>

## 注释<br>
首先需要注意的是**合理使用注释，不要冗余注释，要添加有效注释**<br>

### 注释格式<br>
1. 注释符和注释内容之间应该存在一个空格<br>
2. 注释应该位于代码的上方或右方<br>

## 格式<br>
### 函数大括号统一风格<br>
K&R,1TBS,Allman风格。<br>
有如下几个规范:<br>
```c++
// K&R 风格
void CheckPositive(int x)
{
    if(x>0){
        printf("true");
    }else{
        printf("false");
    }
}

// 1TBS风格
void CheckPositive(int x){
    if(x>0){
        printf("true");
    }else{
        printf("false");
    }
}

// Allman风格
void CheckPositive(int x)
{
    if(x>0)
    {
        printf("true");
    }else
    {
        printf("false");
    }
}
```
注意统一规范即可。<br>

### 行宽上限不超过120<br>
注意行宽。<br>
