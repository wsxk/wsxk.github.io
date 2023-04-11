---
layout: post
tags: [iot]
title: "retrowrite"
date: 2023-2-27
author: wsxk
comments: true
---

- [前言](#前言)
- [retrowrite overview](#retrowrite-overview)
  - [1. Processing](#1-processing)
  - [2. Symbolization](#2-symbolization)
  - [3. Instrumentation Passes](#3-instrumentation-passes)
  - [4. Instrumentation Optimization](#4-instrumentation-optimization)
  - [5. Reassembly(ASM)](#5-reassemblyasm)
  - [思路优点](#思路优点)
- [retrowrite框架实现方法](#retrowrite框架实现方法)
- [和Asan的联合使用](#和asan的联合使用)
  - [design](#design)
  - [implement](#implement)
- [和AFL的联合使用](#和afl的联合使用)
- [retrowrite源码分析](#retrowrite源码分析)
  - [main function](#main-function)


## 前言<br>
这篇文章其实是记录论文`RetroWrite: Statically Instrumenting COTS Binaries for Fuzzing and Sanitization`的内容（真·顶会论文），very牛逼<br>
论文作者提出了一个开源框架`retrowrite`,在[https://github.com/HexHive/retrowrite](https://github.com/HexHive/retrowrite)可以找到他的代码<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230227201328.png)
可以看到有500+star，60+fork，起码还是有不少人尝试使用过了（吐槽某个神秘uSBS<br>
在这篇论文中，作者首先对当前的二进制重写工具进行了一系列评估，并认为它们的效率太差了。于是提出了一个新的二进制重写思路，并使用这个思路写出了新的开源二进制框架，并表示：`我的开源框架很猛，效率比当前的其他框架，效率能加个好几倍！`<br>


## retrowrite overview<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230227202037.png)
这个工具的工作流程被总结成了5步。<br>
即`1.Processing 2.Symbolization 3.Instrumentation Passes 4.Instrumentation Optimization 5.Reassembly`<br>

### 1. Processing<br>
在这个阶段，该框架做了如下几件事:
**分析二进制文件并加载 text & date section，还从中抽取出符号信息和重定位信息。**<br>
**将二进制码用 `linear sweep`方法反汇编成汇编代码**<br>
**在前两步的基础上，进行轻量的分析，构造cfg（直接跳转才能被加边）**<br>

### 2. Symbolization<br>
利用 第一步生成的 cfg 以及 重定位信息 识别在text、data段中的可符号化的常数，并用`assembly labels`来替换他们。<br>
此时已经有了带有`assembly labels`的汇编代码(assembly)<br>
再具体而言，符号化的操作经过了3个步骤<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230227204425.png)
**首先找出 call和jump的指令，并用assembly labels替换它们**<br>
**其次找出 PIC中找到计算PC-relative addresses的指，用assembly labels替换它们**<br>
**最后在data段中，根据重定向条目指向的偏移，用assembly labels替换它们**<br>


### 3. Instrumentation Passes<br>
就是进行一些插桩操作<br>

### 4. Instrumentation Optimization<br>
对现有的插桩进行优化，包括分析插桩并确定需要用到的寄存器数量以及副作用<br>
这一步归结起来主要做2件事<br>
**选择性的保留/恢复 状态变化**<br>
**为插桩来申请寄存器**<br>

### 5. Reassembly(ASM)<br>
再重新将汇编代码转变成二进制代码<br>

### 思路优点<br>
一、不需要将汇编代码再提升一个层次到IR中,（通常这种做法很耗时，而且容易出错<br>
二、轻量，而且可以直接在 现有的反汇编器生成的反汇编代码中 进行操作。<br>
**三、利用relocation、PC-relative addressing以及恢复的控制流，来确定代码中的指针信息**<br>
四、因为不是使用启发式算法，因此没用误报和漏报，说明这个框架是通用的，应用范围很广。<br>


## retrowrite框架实现方法<br>
用`capstone`实现反汇编，用`pyelftools`加载elf文件信息并处理重定向信息。<br>
在`symbolization`步骤中，生成可重新汇编的汇编代码。<br>
为了能够安全的`instrumentation`插桩(即插入之后不影响程序的正常功能运行，有3个问题需要解决）:
**(i) a logical abstraction for analysis and instrumentation passes to operate on, e.g., modules, functions, or basic block level granularity** （即需要一些函数的相关信息等等）。 `对此，解决办法是获得函数的名称等等，对于stripped后的二进制文件，可以采用现有的工具（比如ida）对函数名称进行还原`.<br>
**(ii) working around the ABI to ensure the instrumentation does not break the binary**（围绕应用程序二进制接口进行工作，避免损坏二进制文件）。`为此，需要在汇编层面根据函数的调用约定，传参，返回值，隐式规则等等进行匹配`<br> 
**(iii) automatic register allocation to achieve compiler-like overhead.**（自动进行寄存器的申请，以达到跟编译器一样的开销）`其实最安全的方法是将所有寄存器被使用前都保存下来，但一来需要空间，二来耗时；对此使用intra-function liveness analysis来找到在进行调用时有用的寄存器，将它们的值存储下来（必须是sound的方法，保证所有活寄存器可以被保留。`<br>


## 和Asan的联合使用<br>
原话是这样的`Our goal is to implement a binary version of ASan in RetroWrite that closely resembles and integrates seamlessly with the source-based sanitizer.`.
### design<br>
论文作者认为，将asan应用到二进制文件上的主要问题在于**没有变量、类型、缓冲区长度的信息**，虽然静态分析可以缓解这些问题，但是实用，无法大规模使用，因此作了折中。<br>
下图可以看出ASan和二进制级别的`Asan-retrowrite`的区别。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230228164732.png)
对于栈，只对栈帧进行检测<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230228164836.png)
并且，并不是对所有的函数的栈帧加入`redzone`，而是找到加入`canary`的函数加入`redzone`，并将原本canary的空间留作`redzone`。<br>
在函数返回时，`unpoison`相应区域，对于`longjmp`的指令
不对global变量进行修改（可以看原文，还是存在问题的）<br>


### implement<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230228165622.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230228165640.png)

## 和AFL的联合使用<br>
因为`retrowrite`得到的汇编代码和compiler的类似，而AFL的插桩是直接基于汇编代码进行的。因此可以直接使用`afl-gcc` 生成二进制文件，链接动态库等等。


## retrowrite源码分析<br>
开始分析retrowrite的源码<br>
它的源码因为支持x64和aarch64位的，看起来会比较抽象，因此我把它主要运行代码关于x64的源码扣了出来，方便观看<br>
### main function<br>
```python
import argparse
import json
import tempfile
import subprocess
import os
import sys
import traceback
import importlib
from elftools.elf.elffile import ELFFile
from librw_arm64.util.logging import info

from librw_x64.rw import Rewriter
from librw_x64.analysis.register import RegisterAnalysis
from librw_x64.analysis.stackframe import StackFrameAnalysis
from rwtools_x64.asan.instrument import Instrument
from librw_x64.loader import Loader
from librw_x64.analysis import register

bin = "./heap"
outfile = "heap.s"
arch = "x64"
cache = 1
args = ""

Rewriter.detailed_disasm = True

def load_analysis_cache(loader, outfile):
    with open(outfile + ".analysis_cache") as fd:
        analysis = json.load(fd)
    print("[*] Loading analysis cache")
    for func, info in analysis.items():
        for key, finfo in info.items():
            loader.container.functions[int(func)].analysis[key] = dict()
            for k, v in finfo.items():
                try:
                    addr = int(k)
                except ValueError:
                    addr = k
                loader.container.functions[int(func)].analysis[key][addr] = v

def save_analysis_cache(loader, outfile):
    analysis = dict()

    for addr, func in loader.container.functions.items():
        analysis[addr] = dict()
        analysis[addr]["free_registers"] = dict()
        for k, info in func.analysis["free_registers"].items():
            analysis[addr]["free_registers"][k] = list(info)

    with open(outfile + ".analysis_cache", "w") as fd:
        json.dump(analysis, fd)

def analyze_registers(loader):
    StackFrameAnalysis.analyze(loader.container)
    if cache:
        try:
            load_analysis_cache(loader, outfile)
        except IOError:
            RegisterAnalysis.analyze(loader.container)
            save_analysis_cache(loader, outfile)
    else:
        RegisterAnalysis.analyze(loader.container)

def asan(rw, loader):
    analyze_registers(loader)

    instrumenter = Instrument(rw)
    instrumenter.do_instrument()
    instrumenter.dump_stats()


# load binary
loader = Loader("./heap")

if arch == "x64" and loader.is_pie() == False :
    print("***** The x64 version of RetroWrite requires a position-independent executable. *****")
    print("It looks like %s is not position independent" % bin)
    print("If you really want to continue, because you think retrowrite has made a mistake, pass --ignore-no-pie.")
    sys.exit(1)
if arch == "x64" and loader.is_stripped() == True:
    print("The x64 version of RetroWrite requires a non-stripped executable.")
    print("It looks like %s is stripped" % bin)
    print("If you really want to continue, because you think retrowrite has made a mistake, pass --ignore-stripped.")
    sys.exit(1)

# get sections info
slist = loader.slist_from_symtab() #get all section baseinfo
if not ".gcc_except_table" in slist:
    # if there are no exceptions, we never emulate calls
    Rewriter.emulate_calls = False

loader.identify_imports() 
flist = loader.flist_from_symtab() # before loading data sections: get systable baseinfo
loader.load_functions(flist)
loader.load_data_sections(slist, lambda x: x in Rewriter.DATASECTIONS)

reloc_list = loader.reloc_list_from_symtab()
loader.load_relocations(reloc_list)

global_list = loader.global_data_list_from_symtab()
loader.load_globals_from_glist(global_list)

loader.container.attach_loader(loader)

# symbolize
rw = Rewriter(loader.container, "heap.s")
rw.symbolize()

asan(rw, loader) # address sanitizer
rw.dump()
```

单看主代码还是比较简洁的<br>
实际上也符合论文描述。<br>
首先加载二进制程序，并做一些处理，然后符号化，最后插桩。<br>
至于为什么缺少了重新汇编的功能，因为它重新汇编需要依赖本地的编译环境<br>
之后会着重看一下其中几个类型的实现，然后继续补充<br>