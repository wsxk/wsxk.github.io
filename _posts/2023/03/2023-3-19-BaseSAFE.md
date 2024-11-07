---
layout: post
tags: [fuzz]
title: "BaseSAFE: Baseband SAnitized Fuzzing through Emulation"
author: wsxk
date: 2023-3-19
comments: true
---

- [前言](#前言)
- [简介](#简介)
  - [1. fuzz](#1-fuzz)
  - [2. unicorn](#2-unicorn)
  - [3. Cellular Baseband](#3-cellular-baseband)
- [程序运行逻辑](#程序运行逻辑)
  - [1. afl\_forkserver\_start](#1-afl_forkserver_start)
  - [2. afl\_next](#2-afl_next)
  - [3. afl\_emu\_start](#3-afl_emu_start)
  - [4. afl\_fuzz](#4-afl_fuzz)
- [安全检测实现思路](#安全检测实现思路)
- [harness](#harness)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## 前言<br>
这是取自 `BaseSAFE: Baseband SAnitized Fuzzing through Emulation` 论文的笔记<br>

## 简介<br>
这篇论文将 `fuzz`技术和 `unicorn`技术结合起来，针对 `cellunar baseband`进行安全测试。<br>

### 1. fuzz<br>
不用多说了，大家肯定都耳熟能详<br>

### 2. unicorn<br>
来自`qemu`的分支，github网址在[https://github.com/unicorn-engine/unicorn/tree/7e4754ad008f0dac6d990a7a997a764062b35d04](https://github.com/unicorn-engine/unicorn/tree/7e4754ad008f0dac6d990a7a997a764062b35d04)<br>
`unicorn`从`qemu`出家，当然支持许多架构的模拟运行，于`qemu`不同的是，`unicorn`提供了更方面用户使用的`API接口`，**unicorn暴露了读写内存的函数，以及hook特定地址跳转到回调函数的功能，然而作者还对第三方unicorn-rs bindings进行了拓展**<br>
Unicorn的工作原理和qemu类似<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319141716.png)

### 3. Cellular Baseband<br>
手机除了普通的应用处理器（跑程序）外，还有一种`baseband processor`，<br>
这种处理器用于处理信号的（比如5G信号传来，翻译成手机信息）<br>

## 程序运行逻辑<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319151047.png)
扩展的api如下<br>
### 1. afl_forkserver_start<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319210904.png)

### 2. afl_next<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319210844.png)

### 3. afl_emu_start<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319210818.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319210830.png)

### 4. afl_fuzz<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319210716.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230319210731.png)

## 安全检测实现思路<br>
利用了`unicorn`提供的API以及自扩展的`rs-bindings`，对内存分配进行hook，执行自己的策略<br>

在arm32嵌入式设备中，`memory corruption`和`corrupted memory use`通常是分离的，在这是，`fuzzer`通常开始了下一轮迭代了（重新起一个新的程序），这时会去除所有的bug的信息。<br>
为了解决这个问题，作者插入了一个`drop-in allocator`<br>
> 1. 用 unicorn提供的hooking functionality，直接在JITed code里插入条件检查和回调函数
> 2. 事先分配一个很大的内存，访问时，触发access-hook，分配时会使用自己的分配器，取消access hook，根据分配大小，每个被分配块两边都会挂上一个canary块
> 3. 取消分配时，又挂上access hook，chunksize置为0（独立于结构体之外,因此不会被消除）


## harness<br>
一个简单的harness编写<br>
```python
#../AFLplusplus/afl-fuzz -U -m none -i ./sample_inputs -o ./output -- python3 test_harness.py @@ 
import argparse
import os
import signal
from unicornafl import *
from unicornafl.mips_const import *

BINARY_FILE = "./simple_target.bin"
# Memory map for the code to be tested
CODE_ADDRESS = 0x00100000  # Arbitrary address where code to test will be loaded
CODE_SIZE_MAX = 0x00010000  # Max size for the code (64kb)
STACK_ADDRESS = 0x00200000  # Address of the stack (arbitrarily chosen)
STACK_SIZE = 0x00010000  # Size of the stack (arbitrarily chosen)
DATA_ADDRESS = 0x00300000  # Address where mutated data will be placed
DATA_SIZE_MAX = 0x00010000  # Maximum allowable size of mutated data

# Instantiate a MIPS32 big endian Unicorn Engine instance
uc = Uc(UC_ARCH_MIPS, UC_MODE_MIPS32 + UC_MODE_BIG_ENDIAN)
print("Loading data input from {}".format(BINARY_FILE))
binary_file = open(BINARY_FILE, "rb")
binary_code = binary_file.read()
binary_file.close()

# Apply constraints to the mutated input
if len(binary_code) > CODE_SIZE_MAX:
    print("Binary code is too large (> {} bytes)".format(CODE_SIZE_MAX))
    exit()

# Write the mutated command into the data buffer
uc.mem_map(CODE_ADDRESS, CODE_SIZE_MAX)
uc.mem_write(CODE_ADDRESS, binary_code)

# Set the program counter to the start of the code
start_address = CODE_ADDRESS  # Address of entry point of main()
end_address = CODE_ADDRESS + 0xF4  # Address of last instruction in main()
uc.reg_write(UC_MIPS_REG_PC, start_address)

# Setup the stack
uc.mem_map(STACK_ADDRESS, STACK_SIZE)
uc.reg_write(UC_MIPS_REG_SP, STACK_ADDRESS + STACK_SIZE)

# reserve some space for data
uc.mem_map(DATA_ADDRESS, DATA_SIZE_MAX)

# -----------------------------------------------------
# Set up a callback to place input data (do little work here, it's called for every single iteration)
# We did not pass in any data and don't use persistent mode, so we can ignore these params.
# Be sure to check out the docstrings for the uc.afl_* functions.
def place_input_callback(uc, input, persistent_round, data):
    # Apply constraints to the mutated input
    if len(input) > DATA_SIZE_MAX:
        # print("Test input is too long (> {} bytes)")
        return False

    # Write the mutated command into the data buffer
    uc.mem_write(DATA_ADDRESS, input)

# Start the fuzzer.
uc.afl_fuzz(BINARY_FILE, place_input_callback, [end_address])
```