---
layout: post
title: "python 类型转换 & 调用c动态库的办法 & z3模块 & type hints"
date:   2022-4-8
tags: [knowledge]
comments: true
author: wsxk
---

`PS: 更新于2023-7-7`<br>

- [类型转换](#类型转换)
- [调用c动态库的办法](#调用c动态库的办法)
- [z3介绍](#z3介绍)
  - [z3 安装](#z3-安装)
  - [z3一般流程](#z3一般流程)
- [type hints](#type-hints)


## 类型转换

之所以要记录这么一篇文章，是因为在做ctf题的时候，你得到了答案，往往需要做一个类型的转换（比如你算出来一个大整数，你需要把他转换成字符串可见形式），但是我在类型转换的时候经常出错，不知道python里面内置了哪些转换函数可以给我操作，我自己又不想写（太懒了），最终还是决定记一下转换的方法，让自己以后不要因为转换问题浪费时间


    ord() #输入是一个字符，输出是其对应的ascii码
    chr() #输入是一个数字，输出是其对应的字符

    int() #可以将一个数字（以str的形式出现），转换成真正 的x进制数字，十分方便
    str() #可以将一个真正的十进制数字转换为一个数字（str形式）

    hex() #可以将一个真正的十进制数字转换为一个十六进制数字（str形式）

    int.to_bytes(num,'little') #int表示一个int型变量，可以将一个真正的数字，转换成bytes形式

    byte.hex() # b'\x61\x61\x61\x61'.hex()='61616161' 还是比较方便的
    
    u32() # 都是pwntools里的自带工具
    p32()
    u64()
    p64()

## 调用c动态库的办法<br>

python中有时为了加快运行速度，会选择直接调用c动态库中的函数运行

当然，我会去查找调用c库的方法主要是因为 ctf pwn的需要

常规的调用方法如下

    from ctypes import CDLL  #linux 下
    from ctypes import WinDLL #windows下

    libc = CDLL("DLLpath")

    a=libc.time(0)
    libc.strand(a)
    print(libc.rand())

## z3介绍

Z3是由Microsoft Research开发的高性能定理证明器。(可以帮我们快速解方程)。Z3 在工业应用中实际上常见于软件验证、程序分析等。

由于Z3功能实在强大，也被用于很多其他领域：软件/硬件验证和测试，约束解决，混合系统分析，安全性，生物学（计算机模拟分析）和几何问题。

CTF 领域来说，能够用约束求解器搞定的问题常见于密码题、二进制逆向、符号执行、Fuzzing 模糊测试等。此外，著名的二进制分析框架 angr 也内置了一个修改版的 Z3。（当然了，我们关注的是就是它的自动解方程功能啦）

### z3 安装

    pip install z3

### z3一般流程

    from z3 import *   # 1.导入库

    enc = [BitVec("%id"%i,32) for i in range(4)] # 2.声明变量

    s= Solver() # 3.创建一个约束求解器

    s.add(enc[3]*28096+enc[2]*64392+enc[1]*29179+enc[0]*52366==209012997183893)
    s.add(enc[3]*61887+enc[2]*27365+enc[1]*44499+enc[0]*37508==181792633258816)
    s.add(enc[3]*56709+enc[2]*32808+enc[1]*25901+enc[0]*59154==183564558159267)
    s.add(enc[3]*33324+enc[2]*51779+enc[1]*31886+enc[0]*62010==204080879923831)  # 4.添加条件（就是添加方程组）

    print(s.check())  # 4.求解 如果有解打印的是sat
    print(s.model())  # 得到解

    answer = s.model() # dictionary
    flag = b''
    for i in enc:
        #print(i)
        #print(type(i))
        temp = answer[i].as_long()
        temp = temp.to_bytes(4,"little")
        flag = temp + flag

    print(flag)


## type hints<br> 
不知道大家有没有遇到这样一个情况，在类型A中，使用到了类型B的变量c，然而，在实现类型A时，IDE无法识别到c就是类型B，从而失去了类型补全，导致我们编写代码时出现了困难。<br>

类型补全是一个强大的功能，它可以帮助ide识别变量类型，从而方便你进行补全<br>

```python
from typing import Optional

class B:
    def method_in_b(self):
        print("Hello from class B")

class A:
    def __init__(self, b_instance: Optional[B] = None):
        self.b_instance = b_instance

    def use_b(self):
        if self.b_instance:
            self.b_instance.method_in_b()
        else:
            print("b_instance is not set")

# 使用示例
b = B()
a = A(b_instance=b)
a.use_b()
```