---
layout: post
tags: [iot_dev]
title: "iot dev technology 3"
date: 2023-11-6
author: wsxk
comments: true
---


- [6. 设计硬件抽象层](#6-设计硬件抽象层)
  - [6.1 由eCos \& amp;Android的系统架构谈起](#61-由ecos--ampandroid的系统架构谈起)
  - [6.2 HAL vs. BSP](#62-hal-vs-bsp)
  - [6.3 为什么会需要HAL?](#63-为什么会需要hal)
  - [6.4 HAL是否会增加开发难度?](#64-hal是否会增加开发难度)
  - [6.5 HAL实例](#65-hal实例)
    - [6.5.1 HAL基本设计原则](#651-hal基本设计原则)
    - [6.5.2 HAL的系统架构](#652-hal的系统架构)
    - [6.5.3 驱动程序与系统的沟通机制](#653-驱动程序与系统的沟通机制)
    - [6.5.4 难以抽象化的设备](#654-难以抽象化的设备)
- [7. 菜鸟当自强：软件工程师硬起来　](#7-菜鸟当自强软件工程师硬起来)
  - [7.1 硬件开发流程](#71-硬件开发流程)
  - [7.2 嵌入式软件开发工程师的基本艺能](#72-嵌入式软件开发工程师的基本艺能)
- [8. 做好存储器管理](#8-做好存储器管理)
  - [8.1 动态存储器空间配置](#81-动态存储器空间配置)
  - [8.2 Stack](#82-stack)
    - [8.2.1 Stack 的用途](#821-stack-的用途)
    - [8.2.2 Stack Overflow](#822-stack-overflow)
    - [8.2.3 Stack \& RTOS](#823-stack--rtos)
- [9. 存储器管理（II）：NAND Flash概论　](#9-存储器管理iinand-flash概论)
- [10. 模拟器](#10-模拟器)
- [11. Callback Function](#11-callback-function)
- [12. 用C来实作面向对象的概念 　](#12-用c来实作面向对象的概念-)


## 6. 设计硬件抽象层<br>
请先看下图所示的架构：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231106214622.png)
这个架构同时要支持两种系统（RTOS与Embedded Linux），但我们又希望driver可以共享，不要有两套，否则maintain将会是个大麻烦，所以在driver上再加了一层HAL，所谓HAL就是将硬件抽象化，Linux可以基于HAL再往上开发自己格式的driver。<br>
HAL的另一个好处是就算硬件配置做了修改，也只需要修改driver，两方系统都不需要大更动。<br>
### 6.1 由eCos & amp;Android的系统架构谈起<br>
般通用性的嵌入式操作系统，如VxWorks、μCOS II等，可能应用在截然不同的产品上，硬件配置南辕北辙，所以很难去定义通用的HAL;但如果我们的产品线定义颇为明确的话，若希望我们的系统能够具有可扩展性，可以用较少的effort来执行未来的项目，那么，HAL就不可或缺。<br>
Open Source的嵌入式操作系统——eCos，它的系统架构就明确定义了HAL这一层<br>
我们来看另一个例子。最近火红的手机系统Android，因为Android的应用锁定在手机上，且智能手机的硬件架构差异有限，所以可以看到Android的HAL中就包含了许多硬件设备。<br>
举例来说，不同的手机制造商可以选用不同的GPS硬件模块，只要能按照Android HAL中对GPS设备所定义的行为模式（如送出位置信息的格式），并包装出指定的API，则其他程序都不需要修改，内置的导航应用程序应该就可以正常运行。<br>
### 6.2 HAL vs. BSP<br>
HAL之前已经提到很多了，现在具体说说`BSP`的定义<br>
```
■　BSP（Board Support Package）：BSP介于主板硬件和操作系统间，其功能与PC主机板上的BIOS相类似，主要功能为完成硬件初始化，并切换到相应的操作系统。

■　BSP是相对于操作系统而言的，不同的操作系统会对应于不同定义形式的BSP。例如，某一CPU上的VxWorks BSP和Linux BSP，尽管实现的功能一样，但写法和接口定义是完全不同的。

■　BSP单纯只支持某个硬件板子的配置，无法用于其他板子、CPU或操作系统。举例来说：微软定义BSP是由Boot-loader、OAL（OEM Adaptation Layer）、Device Driver及Run-Time Image Configuration File这几个组件所构成。
```
**所以BSP和HAL是不一样的东西，应用时机也不同。BSP是针对某个板子的硬件配置，进行驱动程序套件开发，并包装出某种OS的接口，让使用者拿到板子后，可以很快地将OS与应用程序移植到这个板子。Linux、VxWorks、Microsoft Windows Mobile都有BSP这样的说法**<br>
相较于HAL，BSP是一专用于某个板子、某种系统的一组驱动程序套件，但HAL是一种概念、一组作为系统标准的API。嵌入式系统基本上是个分层架构，上层通过下层提供的API与下层沟通，而分层的好处就在于只要API不变，上层不需要知道下层的实现方式与细节，就算下层模块整个切换或全部改写，上层程序都不需要修改.<br>
### 6.3 为什么会需要HAL?<br>
很简单，为了减少未来开发的成本，建立商业竞争的领先<br>
假设第一代产品大卖，此时竞争对手肯定会争相研发同样的产品，为了保持自身的领先，此时当然需要投入资源到第二代产品的开发当中，此时，如果第一代产品设计了HAL，那么进行第二代产品开发说，会大大减少开发成本和时间耗费。<br>
### 6.4 HAL是否会增加开发难度?<br>
直接说结论：不会<br>
通常我们实现HAL的流程如下:<br>
```
Step01：定义HAL的规模（Scope）。根据产品特性，分析上层的系统与应用程序需要哪些硬件功能，这些需求就是HAL必须要包含的基本模块，然后再根据硬件配置，分析是否仍有目前系统没用到的硬件功能，这些功能可能在下一代产品就会被用到，我们可为其设计相应HAL模块的API，但在本项目内可先不implement。

Step02：定义HAL API。如果可以的话，最好从硬件到应用程序的开发小组，都要派人员参与HAL设计工作，否则至少要有资深的系统与固件工程师参与，不能仅由一个部门闭门造车。一般定义HAL API步骤为：
    □　系统工程师说明系统需求，包含系统对硬件事件的处理方式。
    □　固件工程师根据硬件功能，提供第一版的HAL API定义文件。
    □　系统工程师协同固件工程师，逐一review HAL API。
    □　在HAL API立项前，必须针对项目中的所有软件工程师做详细报告，收集建议，并做必要的修正。

Step03：由固件工程师负责实现所有HAL的功能，并执行每个模块的单元测试。

Step04：固件工程师release第一版HAL library，由系统工程师负责整合测试，若有需要。各个部门随时可以提出对HAL API增修的建议。
```
HAL的设计立项后，固件工程师就是照着文件去implement，而系统工程师就被规范只能用HAL API来控制硬件，并不会增加工程师们在实作阶段的工作量。就算系统架构中没有HAL，一般driver的开发流程和上述步骤也差异不大吧.<br>
**没有人能够一开始就预知未来，所以不可能直接开发出完善的HAL，只要后续能继续补上就可以了**<br>

### 6.5 HAL实例<br>
#### 6.5.1 HAL基本设计原则<br>
```
■　往前兼容（新版HAL的API可以变多，但不能变少）。

■　HAL旨在对硬件做抽象化，不应与任何系统有直接关系。若OS对driver有特殊的需求（如Linux有制式的driver写法），应由系统团队根据需求自行在HAL之上，再往上包一层high level driver。

■　Driver模块化，尽量降低driver之间的耦合程度。这个原则相当重要，因为我们设计HAL的主要目的之一，就是未来项目的硬件配置可能改变，而我们只要切换驱动程序即可。若driver间耦合度过高，则系统的可移植性与HAL的优势将大打折扣。

■　每个driver分别设计，但章节内的结构必须保持一致，应该包含以下信息。
    □　数据结构
    □　API
    □　控制流程或状态机
    □　会产生的硬件事件，以及事件传递流程
    
■　每个driver模块应该尽量包装出以下的API，以保持所有driver模块的API风格一致，使得系统在使用HAL API时，比较不会发生误用的状况。  
    □　Open（）：执行该设备的初始化工作（包含硬件初始化、取得driver所需之buffer、数据结构或配置的设定）。
    □　Enable（）：driver开始运行。
    □　Disable（）：driver暂停运行。
    □　Close（）：关掉硬件设备，还回已配置的buffer，重设（Reset）driver的数据结构或配置。
    □　Set_Power_mode（）：设定该设备的power mode（如full run、idle、sleep mode），即便有些设备无法实现这么多种的power mode，甚至有些设备根本不需做电源管理，HAL设计者还是可以考虑为此设备包出空的Set_Power_mode（）函数，以保持所有driver API的一致性。
    
■　driver与power manager的配合：driver仅提供电源管理的mechanism，而电源管理的policy由系统的power manager统一管控，即HAL中的每个driver模块都必须配合power manager的需求，例如都必须实现Set_Power_mode（）这样的API。

■　HAL的每个driver模块都必须遵守共同命名规则与API style。

■　统一定义HAL的系统配置（Configuration）：这些配置必须开放给系统使用，使得本项目不需要的driver模块或功能不会被link进来.
```

#### 6.5.2 HAL的系统架构<br>
刚刚提到driver之间的耦合程度越低越好，但driver间难免也是会有阶层关系，例如I2C driver与FM Module Driver（可播放FM广播的芯片，与其他必备零件整合在一个比拇指小的板子里），主控IC通过I2C对其FM Module下命令，FM Module就会根据设定开始接收与播放FM频道。以HAL设计的角度来看，虽然I2C在这个项目只用来控制FM Module，但其他项目可能会用I2C来控制其他芯片，所以I2C必须是独立的driver，而FM Module Driver则通过调用I2C的API来运行。<br>
就如同系统设计会做出各个模块之间的关系图，包含模块间的调用关系、参数传递的规范、共享全局变量的规则、模块间运行的流程与顺序等，我们可以把HAL当作一个独立的子系统，所以在进入个别driver的设计阶段前，HAL设计文件必须比照系统设计文件，事先定义出所有driver间的关系，作为设计个别driver时的规范。<br>

#### 6.5.3 驱动程序与系统的沟通机制<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231107211512.png)
```
■　HAL API：系统可以直接调用HAL开放的API。

■　ISR & Hardware Event：当硬件产生中断时，相应的ISR会被执行；当ISR处理完毕后，可将硬件事件抽象化，包装成硬件事件，往上层系统传递。以低电压事件为例，驱动程序自硬件读取的是AD值，必须将其换算为电压值（此举即为抽象化），并对上层传递此硬件事件。
    □　驱动程序可以把一个全局变量当作flag，当上层程序发现此flag的值改变了，即表示发生了某硬件事件。这种做法的时效性较差，且必须注意Critical Section的保护。
    □　在ISR内调用系统的功能，如send_message（）、wakeup_task（）等，这种做法是比较普遍的做法，但此举破坏了HAL必须与上层系统无关的规范。
    □　较好的做法是HAL不决定传递硬件事件的机制，改为提供API，让系统可‘注册’用以传递硬件事件的函数（称为callback function），ISR会在适当时机调用这些callback function，至于系统如何处理硬件事件，ISR并不会知道。

■　Callback：HAL提供注册callback function的API，并明确说明调用此callback function的时机。系统只要传入function pointer，HAL即会在上述时机调用这些callback function。例如上面所说的实时处理硬件事件，或是当HAL准备关机时，会去调用系统预先注册的callback fimction，让系统有机会做一些处理（如存储重要信息或目前的执行状态）。
```
#### 6.5.4 难以抽象化的设备<br>
例如某颗CPU的clock可以调整为3.768 K、12 MHz、24 MHz、48MHz4 个选项，那我们的 HAL API——hal_set_CPU—clock()的参数要如何设计？若下个项目换了较快的CPU，它的clock的设定值根本不一样，假设是3.768 K、16 MHz、32 MHz、64 MHz，这时候HAL API不就非改不可了吗？<br>
**为了解决这个问题，需要再往高端抽象化一层，如下图所示：**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231107212201.png)

## 7. 菜鸟当自强：软件工程师硬起来<br>　
### 7.1 硬件开发流程<br>
固件工程师应该参与`hardware review meeting`<br>
好处如下<br>
```
■　可了解电路设计的主体，在设计阶段就可发现固件编写时必须特别注意的项目。

■　若对某外部设备的控制方式不是很有把握，可以在电路设计时，请硬件人员预留其他设计方案（举例来说，驱动程序可以用SPI接口控制SD卡，也可以外接一颗SD card Controller。前者电路简单、成本低、速度慢；后者则刚好相反。在设计阶段，系统工程师不见得有把握使用SPI接口就可以符合产品应用的需求，此时可以请硬件工程师将两个方案都做到target board上）。

■　不仅仅只有硬件工程师才可与硬件原厂联系，固件工程师也应该尽可能寻求原厂的协助，并多与硬件工程师交换意见。

■　事前的讨论，胜过事后的争执。
```
硬件工程师需要配合固件工程师，固件工程师也应该提供硬件工程师一些帮助：**除了板子验证阶段的测试程序外，软件工程师还必须额外提供一些程序给硬件工程师，通常这些程序可称为系统的工程模式（Engineering Mode）或测试模式（Self-Test Mode），主要用途就是快速地检测产品规格，并检查硬件是否可正常运行。**<br>

### 7.2 嵌入式软件开发工程师的基本艺能<br>
虽然大部分硬件工作主要还是交给硬件工程师来干，但是软件开发工程师也至少会一些硬件技能:<br>
```
1. 烙铁： 焊板子
2. 看懂电路图： 电路图是一种语言，它沟通的方式是利用简单的线段和符号去叙述组件之间的关系，电路图里包含各个组件及其PIN脚的名称，及所有组件之间的连接关系。
3. 电源供应器： 这你得会用吧
4. 三用电表： 高中物理就用过
5. 示波器： 大学物理有用过。 
trigger（触发）功能是使用示波器一定要会用的技巧，示波器预设一直从探针中收集信号，即便探针没有点在有意义的测点也一样，一旦启动了示波器的Trigger功能，其设定决定了当信号发生某种变化时，示波器才会再开始显示波形，并记录一段时间内的波形变化。
```

## 8. 做好存储器管理<br>
### 8.1 动态存储器空间配置<br>
一般的嵌入式设备存储器由如下几个部分组成：<br>
```
■　CPU Internal SRAM——CPU 内部有两块 RAM，分别是 40K Bytes 与 16K Bytes。作为LCD控制器的Video RAM，或用来加速程序模块的执行性能。

■　外部NOR Flash——1M Bytes。用来放置程序与data，且程序可以在NOR Flash中直接执行，这块存储器也可以用Mask ROM取代。

■　外部SRAM——512K Bytes。执行时期放置程序的所有变量（data段与bss段）、程序执行时所需的Stack Memory、以及动态配置存储器（Dynamic Allocation Memory）或缓冲区（Buffer）。

■　SD Card。我们的系统可支持SD Card，额外的大批数据可以存储在SD Card中。
```
但是应用开发工程师不需要知道这些细节,他们只需要知道地址范围即可：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231109211334.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231109211411.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231109211423.png)
此时，对于一个`100k大小的空间`，可以放置的位置如下：
```
■　放置在rodata区（Read-Only Data Section）,程序中只要用数组的名称即可操作这批数据。

■　如果上述数组不定义为const，则这批数据会被连接到data区（有初值的全局变量）。data区的数据也会占据可执行文件的空间，在执行时期会被复制到RAM中，所以程序可以改变这个数组的值。这个方法会加快操作这批数据的性能（RAM速度一定比Flash快），缺点是执行时期时同一批数据分别在RAM与Flash占据一份空间。如果没有改变内容与加速的需求，这么做实在是浪费空间。

■　当NOR Flash或Mask ROM空间所剩不多，但RAM的空间还充裕时，有时候我们会考虑将这批数据压缩。
```

### 8.2 Stack<br>
以前说过，一个程序的分布应该是如下:
```
■　程序段（text段与rodata段）：可直接在ROM或Flash中执行，也可将某个模块传输到速度较快的RAM里执行。

■　有初始值的全局变量（data段）：会占据可执行文件空间，执行时期必须将其从ROM或Flash传输到RAM。

■　没有初始值的全局变量（bss段）：不会占据可执行文件空间，执行时期必须将该区段的内容全部设为0。

■　Stack（堆栈）。
```
#### 8.2.1 Stack 的用途<br>
```
■　在执行“call”指令时，会将返回地址存储（Push）到目前SP寄存器指到的位置（SP始终指到Stack的顶端）。

■　当执行“ret”（Return）指令时，会自目前Stack的顶端取回（Pop）返回地址。

■　当中断产生时，中断控制器会将发生中断的地址（返回地址）以及PSR（状态寄存器）存储（Push）到Stack里。

■　当ISR执行“iret”（Interrupt Return）指令时，CPU会从目前Stack顶端Pop出PSR与被中断的地址。

■　我们都知道Stack这个数据结构特性就是先进后出，CPU提供“push”与“pop”指令供程序员操作Stack，最常见的用途就是暂时将某些存储寄存器的值存在Stack中，在某些动作后，再‘依次’取回这些寄存器的值。此外，“push”与“pop”的动作必须是对称的，push多少东西到Stack，就必须记得依次pop多少东西出来，否则下次存取Stack时就数据就会乱掉。

■　Local变量会存储在Stack中，如果应用程序工程师不知道可以用的Stack有多大，一不小心就会把Stack用爆了。此外还要注意的是，用于嵌入式系统的编译器通常不会产生‘将局部变量的值设为0’的code，也就是说，一旦程序员没有明确指定局部变量初值的话，局部变量的初值可能是任何的值。如果程序员疏忽将没给初值的局部变量拿来使用，则结果自然是无法预测的.

■　函数调用时，返回地址当然会根据目前SP寄存器，存储在Stack中，这是CPU的机制，C语言的函数自然也是如此运行。

■　函数调用时参数的传递可以通过寄存器，也可以通过Stack，端视编译器的策略而定。通常如果参数个数不多的话，编译器会倾向用寄存器来传递参数，反之，则把参数存在Stack 中。
```

#### 8.2.2 Stack Overflow<br>
老生常谈了。<br>

#### 8.2.3 Stack & RTOS<br>
我们从RTOS的角度来看Stack Memory的用途，一般用于嵌入式系统的CPU都不会具备虚拟存储器的功能，也就是说系统无法像Windows或Linux 一样为每个程序配置独立的虚拟地址空间，在这样的硬件限制下，RTOS要想实现多任务，其实靠的是`Multiple-Task`<br><br>
在Windows或Linux上的多任务可分为两种，一种是`Multiple-Process`，另一种是`Multiple-Thread`。每一个Process有自己独立的地址空间，而且每一个Process内可以建立多个Thread，所以process内的所有Thread则位于同一个地址空间。<br>
举个简单的例子来说明地址空间的概念。可以把process想象成不同的可执行文件（.exe），因为不同的process有独立的地址空间，假设不同的process都去存取同样的地址（如0×100000），实际上，操作系统会将其mapping到不同的物理地址，所以绝对不会互相冲突。至于同一个process内的不同Thread，因为共享地址空间，所以不同Thread存取同样的地址时就可能会引发冲突。因为Thread没有自己的地址空间，所以Thread之间切换的复杂度远比process间的切换低很多，而且由于多个Thread共享地址空间，所以Thread之间的通信相对简单。总之，只要做好`critical section`保护，Thread的性能较好、程序编写较简单，却同样可以达到多任务的效果，这也是目前`Multiple-Threading`程序设计方法广泛流行的主要原因。<br>
**用于嵌入式系统的RTOS的多任务功能就是上述的Multiple-Thread，通常嵌入式操作系统的书或RTOS的网站会把这个功能称为Multiple-Task，其实是相同的东西，只是名词差异而已。**总之，就是所有的执行单位不论称之为Thread或task，都共享同一个地址空间<br>
可以看看多任务的代码：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231110203150.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231110203216.png)

## 9. 存储器管理（II）：NAND Flash概论<br>　
## 10. 模拟器<br>
## 11. Callback Function<br>
附录B

## 12. 用C来实作面向对象的概念<br> 　
附录C