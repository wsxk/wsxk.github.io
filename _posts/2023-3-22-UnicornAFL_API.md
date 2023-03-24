---
layout: post
tags: [iot]
title: "UnicornAFL Harness"
author: wsxk
date: 2023-3-22
comments: true
---


- [Unicorn](#unicorn)
  - [Unicorn VS qemu](#unicorn-vs-qemu)
- [UnicornAFL install](#unicornafl-install)
- [Unicorn API](#unicorn-api)
- [实践](#实践)


## Unicorn<br>
基于qemu的另一个开源项目。<br>
现在变成了基础设施之一。像 angr，radare2都集成了Unicorn框架<br>
### Unicorn VS qemu<br>
总得来说，unicorn专注于CPU的指令，不提供qemu那样对计算机其他设备的模拟。<br>
优势还是有很多的:<br>

> 1. 框架：容易集成
> 2. 灵活：即使没有上下文信息，即便没有文件格式（ELF），也能模拟运行，使用于dump出来的二进制片段
> 3. 插桩： 提供qemu没有提供的动态插桩，能够自定义插桩技术
> 4. 线程安全： qemu一般只能运行一个cpu，unicorn可以同时运行多个
> 5. bindings： 说白了就是提供了各种语言的API接口，方便开发
> 6. 轻量： 看大小就明白了，你安装qemu要装半天，unicorn很快就搞定了
> 7. 安全： 因为模拟的方面少（就cpu），所以攻击面小（😀


## UnicornAFL install<br>
首先download`AFL++`,网址在[https://github.com/AFLplusplus/AFLplusplus](https://github.com/AFLplusplus/AFLplusplus)<br>
然后去unicorn_mode目录下运行`./build_unicorn_support.sh`<br>
[https://github.com/AFLplusplus/AFLplusplus/tree/stable/unicorn_mode](https://github.com/AFLplusplus/AFLplusplus/tree/stable/unicorn_mode)

## Unicorn API<br>
有一些API需要了解<br>
网址在这里[https://github.com/kabeor/Unicorn-Engine-Documentation/](https://github.com/kabeor/Unicorn-Engine-Documentation/)<br>
自行翻阅


## 实践<br>
写了个程序
```c
#include <stdio.h>
#include <stdlib.h>

int main(){
    char * a = malloc(20);
    printf("char addr:%p\n",a);
    scanf("%s",a);
    printf("%s\n",a);
}
```
编译程序后
写harness如下<br>
```python
import argparse
import os
import signal
import sys
from unicornafl import *
from unicornafl.x86_const import *

DEBUG = 0
if DEBUG:
    def log_to_file(message):
        print(message)
else:
    def log_to_file(message):
        with open("harness_output.txt", "a") as f:
            f.write(message + "\n")


BINARY_FILE = "./malloc_test"
# Memory map for the code to be tested
CODE_ADDRESS = 0x00000000  # Arbitrary address where code to test will be loaded
CODE_SIZE_MAX = 0x00010000  # Max size for the code (64kb)
STACK_ADDRESS = 0x00100000  # Address of the stack (arbitrarily chosen)
STACK_SIZE = 0x00010000  # Size of the stack (arbitrarily chosen)
DATA_ADDRESS = 0x00201000  # Address where mutated data will be placed
DATA_SIZE_MAX = 0x00010000  # Maximum allowable size of mutated data

# maintain allocator
heap_addr = DATA_ADDRESS
heap_size = 0


uc = Uc(UC_ARCH_X86,UC_MODE_64)
log_to_file("Loading file from {}".format(BINARY_FILE))
binary_file = open(BINARY_FILE,'rb')
binary_code = binary_file.read()
binary_file.close()
if len(binary_code)> CODE_SIZE_MAX:
    log_to_file("file too large")
    exit()

#code
log_to_file("mmap code")
uc.mem_map(CODE_ADDRESS,CODE_SIZE_MAX)
uc.mem_write(CODE_ADDRESS,binary_code)
#rip
start_address = CODE_ADDRESS+0x73A
end_address = CODE_ADDRESS + 0x792
uc.reg_write(UC_X86_REG_RIP,start_address)
#stack
uc.mem_map(STACK_ADDRESS,STACK_SIZE)
uc.reg_write(UC_X86_REG_RSP,STACK_ADDRESS+STACK_SIZE)
#data
uc.mem_map(DATA_ADDRESS,DATA_SIZE_MAX)

#hook
log_to_file("add hook func")
def hook_malloc(uc, address, size, user_data):
    #log_to_file(">>> hook instruction at 0x%x, instruction size = 0x%x" % (address, size))
    uc.reg_write(UC_X86_REG_RAX, heap_addr)
    uc.reg_write(UC_X86_REG_RIP,address+size)
    heap_size = uc.reg_read(UC_X86_REG_EDI)
    #log_to_file("heap_size: {}".format(heap_size))
    #log_to_file("set protect")
    #uc.mem_protect(heap_addr,heap_size,UC_PROT_READ|UC_PROT_WRITE)
    #uc.mem_protect(heap_addr+heap_size,0x1000,UC_PROT_NONE)
uc.hook_add(UC_HOOK_CODE,hook_malloc,begin=CODE_ADDRESS+0x747,end=CODE_ADDRESS+0x747)

def hook_scanf(uc, address, size, user_data):
    #log_to_file(">>> hook scanf at 0x%x, instruction size = 0x%x" % (address, size))
    uc.reg_write(UC_X86_REG_RIP,address+size)
    array = uc.mem_read(heap_addr,heap_size)
    #log_to_file("input content: {}".format(array))
uc.hook_add(UC_HOOK_CODE,hook_scanf,begin=CODE_ADDRESS+0x77b,end=CODE_ADDRESS+0x77b)

def hook_printf(uc, address, size, user_data):
    #log_to_file(">>> hook printf at 0x%x, instruction size = 0x%x" % (address, size))
    uc.reg_write(UC_X86_REG_RIP,address+size)
uc.hook_add(UC_HOOK_CODE,hook_printf,begin=CODE_ADDRESS+0x763,end=CODE_ADDRESS+0x763)

def hook_puts(uc, address, size, user_data):
    #log_to_file(">>> hook puts at 0x%x, instruction size = 0x%x" % (address, size))
    uc.reg_write(UC_X86_REG_RIP,address+size)
    #log_to_file("start read heap")
    data=uc.mem_read(heap_addr,30)
    log_to_file("read_data:{}".format(data))
    length = 0
    for i in range(len(data)):
        log_to_file("{}".format(data[i]))
        if(data[i]!=0):
            length+=1
            continue
        break
    log_to_file("len data: {}".format(length))
    if length>20:
        os.abort()
        
uc.hook_add(UC_HOOK_CODE,hook_puts,begin=CODE_ADDRESS+0x787,end=CODE_ADDRESS+0x787)

# execute
log_to_file("start execute")
def place_input_callback(uc, input, persistent_round, data):
    # Apply constraints to the mutated input
    if len(input) > DATA_SIZE_MAX:
        # log_to_file("Test input is too long (> {} bytes)")
        return False
    #log_to_file("input data: {}".format(input))
    #log_to_file("input len: {}".format(len(input)))
    uc.mem_write(DATA_ADDRESS, input)

uc.afl_fuzz(BINARY_FILE, place_input_callback, [end_address])
# sudo ../AFLplusplus/afl-fuzz -U -m none -i ./sample_inputs -o ./output -- python3 harness_malloc_test.py @@ 
```


