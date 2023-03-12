---
layout: post
tags: [iot]
title: "ARM GNU TOOLTRAIN tutorial"
date: 2023-3-11
author: wsxk
comments: true
---


- [如何为microcontroller制造二进制文件](#如何为microcontroller制造二进制文件)
  - [1. Compiler](#1-compiler)
  - [2. Linker](#2-linker)
  - [3. Locator](#3-locator)
  - [Startup Code](#startup-code)
- [Binary production for PCs vs microcontrollers](#binary-production-for-pcs-vs-microcontrollers)
- [Loading and Debugging](#loading-and-debugging)
- [GNU ARM Toolchain](#gnu-arm-toolchain)
- [Tools to Produce The Binaries](#tools-to-produce-the-binaries)
- [Tools to Help Debug Code](#tools-to-help-debug-code)
- [总结](#总结)
- [参考](#参考)


## 如何为microcontroller制造二进制文件<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230311151006.png)<br>

### 1. Compiler<br>
将源代码(`c/c++`)转换成`obj file`，即将**高级语言转换成机器可执行的低级语言**<br>
值得一提的是，因为我们是要在pc平台上生成嵌入式设备运行的程序,所以我们使用的是**交叉编译器**，交叉编译器与普通编译器的区别是，**普通编译器用于在本地生成运行于本地平台的可执行代码，交叉编译器用于在本地生成目标平台的可执行代码**<br>

### 2. Linker<br>
在第一阶段生成的`obj`文件中，通常分成三个部分，`.text,.data,.bss`<br>
**.text段存储代码部分**<br>
**.data段存储已经初始化的全局变量**<br>
**.bss段存储未初始化的全局变量**<br>
众所周知，高级代码语言允许我们将代码分成不同的模块，***Linker的作用就是将各个模块的可执行文件合成一个最终的文件***<br>
方法也很简单，同样.text段的代码逢在一起，同样.data段的数据缝在一起，同样.bss段的数据缝在一起<br>

### 3. Locator<br>
因为不同类型的microcontroller都有不同的内存分布，来一个经典的:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230311191957.png)
因此Locator的作用在于重新编排可执行文件的布局，使得该程序可以在microcontroller上运行，具体而言，**Locator需要正确的标记.text段，.data段，.bss段的地址，以便一但所有的内容加载到microcontroller的flash中，.data段的地址可以刷到RAM中。**<br>

### Startup Code<br>
一个microcontroller的项目通常都有一个称作`startup code`的东西，通常该文件名称位`startup.asm`,`startup.c`,`crt0.s`，一但微控制器通电，它会往特定内存处运行特定的代码（通常是0x00000000处，即运行startup code），从通电到运行main函数之间，运行的都是start up code。**The duty of the startup code is to take the machine from power-on point to the point where the main() function starts executing the application code.**<br>
在通电和运行main函数之间，startup code做了如下几件事情

    1. initializes the important peripherals
    2. the initialized global variables are copied over from the flash to the RAM. (we cannot do this while loading the code to the microcontroller because the RAM is volatile memory!)
    3. The stack and heap are initialized on the free space of the RAM
    4. main() is called

**During the locating process this startup code must be placed at address 0x0000 and all the sections must be labeled with correct addresses so that the microcontroller can do the rest!**<br>

## Binary production for PCs vs microcontrollers<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230312004359.png)<br>

## Loading and Debugging<br>
加载microcontroller程序到主板的flash上的过程叫做`flashing`.要想调试主板上的程序，我们需要一个主板上的特殊外设，名叫`debug controller`，用得到的比较著名的两个协议是是`SWD`和`JTAG`<br>
但是`SWD`和`JTAG`协议相当于一门信号语言，PC端无法识别这种信号，所以为了能够让PC端识别这些协议，`USB Debug adapters`出现了，**USB Debug adapters take data through USB signals and convert them into microcontroller readable JTAG/SWD signals.**<br>

**Thus as we send this binary from the computer’s USB port via debug adapters, the microcontroller’s debug controller peripheral receives it and stores it onto the flash memory**<br>

在调试代码时，**As we click these buttons, special instructions are sent to the debug controller which in turn controls the processor’s execution state.**<br>

## GNU ARM Toolchain<br>
arm-none-eabi<br>
## Tools to Produce The Binaries<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230312112339.png)
其中，1 主驱动程序，可以compile-link-locate一条龙<br>
2是单纯的compile的部分，将汇编语言转换成obj<br>
3是链接器<br>
4是一个特殊的工具:<br>
**There are several formats an object file can be produced in. Popular formats include Extended Linker Format (.elf format) and Common Object File Format (.coff). But these formats are usually for running binaries on PCs and they contain some extra information about the binary.<br>For microcontrollers, the binaries are usually tightly packed without any extra metadata. objcopy is the tool responsible for taking the elf or coff binaries and pack them in a way that can be flashed onto the microcontroller!**

## Tools to Help Debug Code<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230312113153.png)


## 总结<br>
> 1. 用arm-none-eabi系列工具生成microcontroller可执行的程序
> 2. 使用arm-none-eabi-gdb（debug server）进行调试
> 3. OpenOCD会将可执行程序和gdb的调试命令 进行转换，发送给USB debug-adapter
> 4. USB debug-adapter将USB信号转换为JTAG/SWD 信号或相反
> 5. 开发板上的 debug-controler接收到信号后，进行相应的操作，并进行反馈



## 参考<br>
[https://embeddedinventor.com/a-complete-beginners-guide-to-the-gnu-arm-toolchain-part-1/](https://embeddedinventor.com/a-complete-beginners-guide-to-the-gnu-arm-toolchain-part-1/)<br>