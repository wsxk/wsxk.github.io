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
  - [detailed STM32 memory layout](#detailed-stm32-memory-layout)
- [Binary production for PCs vs microcontrollers](#binary-production-for-pcs-vs-microcontrollers)
- [Loading and Debugging](#loading-and-debugging)
- [GNU ARM Toolchain](#gnu-arm-toolchain)
- [Tools to Produce The Binaries](#tools-to-produce-the-binaries)
  - [elf axf coff的区别](#elf-axf-coff的区别)
- [Tools to Help Debug Code](#tools-to-help-debug-code)
- [总结](#总结)
- [参考](#参考)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


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
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230311191957.png)<br>

因此Locator的作用在于重新编排可执行文件的布局，使得该程序可以在microcontroller上运行，具体而言，**Locator需要正确的标记.text段，.data段，.bss段的地址，以便.text段的内容加载到microcontroller的flash中，.data段、.bss段的内容刷到RAM中。**<br>

### Startup Code<br>
一个microcontroller的项目通常都有一个称作`startup code`的东西，通常该文件名称位`startup.asm`,`startup.c`,`crt0.s`，一但微控制器通电，它会从特定内存处加载复位向量（通常是0x00000000处，即startup code的代码起始地址），从通电到运行main函数之间，运行的都是start up code。**The duty of the startup code is to take the machine from power-on point to the point where the main() function starts executing the application code.**<br>
在通电和运行main函数之间，startup code做了如下几件事情

    1. initializes the important peripherals
    2. the initialized global variables are copied over from the flash to the RAM. (we cannot do this while loading the code to the microcontroller because the RAM is volatile memory!)
    3. The stack and heap are initialized on the free space of the RAM
    4. main() is called

**During the locating process this startup code must be placed at address 0x0000 and all the sections must be labeled with correct addresses so that the microcontroller can do the rest!**<br>

### detailed STM32 memory layout<br>
还是这张经典的图。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230311191957.png)<br>
我们需要明确一点：因为stm32是32位arm程序，可以寻址的内存范围起始是4GB，然而实际上一个stm32微控制器是没有那么多内存空间的（一个100+的板子，要4gb的内存，你怕是想太多了😊）<br>
现在我们要做的是仔细思考各个块之间到底放了些什么：<br>
**1. Code**: 这该地址范围的前16字节包含复位向量表，其中包括初始堆栈指针和复位向量（复位向量指向startup code的代码起始位置）,还有比较常用的是0x0800 0000开始的flash（或者rom）区间，startup code和 实际固件逻辑的代码都位于flash中。**注意，此时data段和bss段在flash中也有备份，需要通过startup code拷贝到RAM中（因为flash和rom是只读的，需要拷贝到可读写的RAM中）**<br>
**2. SRAM**: 这个地址空间中，从低到高分布着 data段，bss段，heap段，stack段。**注意：heap段向上增长，stack段向下增长，如果使用不慎，heap和stack是有可能重合的**<br>
**3. Peripherals** :这个范围内的地址主要用于访问和控制外设，如GPIO、串行通信接口、定时器等。采用MMIO的方式。通过将外设寄存器映射到这个区域，软件可以像访问内存单元一样访问这些寄存器，从而实现对外设的控制和配置。这种内存映射I/O方法简化了硬件和软件之间的接口，提高了处理器和外设之间的通信效率。**注意，DMA控制器的相关配置也放在这里**<br>
**4. internal peripherals**: 其实这也是控制外设相关的部分，但是这个区间中主要包含与ARM Cortex-M内核相关的功能和寄存器，这些功能通常不是特定于STM32的，而是与ARM Cortex-M系列处理器共享的.<br>

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

### elf axf coff的区别<br>
AXF (ARM Executable Format)：AXF 文件是专为 ARM 处理器设计的文件格式。它主要用于嵌入式系统和实时操作系统（RTOS）。AXF 文件格式基于 ELF，因此具有 ELF 文件的大部分特性。它支持在 ARM 架构上进行地址空间随机化（ASLR），并提供了一种描述程序执行的方式，包括代码段、数据段和符号表等。<br>
ELF (Executable and Linkable Format)：ELF 文件是一种通用的可执行文件、可链接文件和可重定位文件格式。它在许多不同的操作系统中被广泛使用，例如 Linux，FreeBSD，Solaris 等。ELF 文件包含了程序的二进制代码、数据、符号表和其他信息，以便在链接、加载和执行过程中使用。ELF 文件格式支持多种处理器架构（如 x86，x86-64，ARM，MIPS 等）和多种操作系统。<br>
COFF (Common Object File Format)：COFF 是一种较旧的可重定位文件格式，主要在微软的 Windows 系统和一些 Unix 系统（如 AIX）中使用。COFF 文件包含程序的二进制代码、数据、符号表等信息。与 ELF 相比，COFF 文件格式具有较低的可扩展性和灵活性。微软 Windows 系统已将其替换为更先进的 PE（Portable Executable）格式。<br>
**需要注意的是，直接烧录进嵌入式设备的通常是bin文件or hex文件**<br>

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