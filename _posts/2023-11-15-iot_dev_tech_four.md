---
layout: post
tags: [iot_dev]
title: "iot dev technology 4：NAND Flash & 模拟器 & call back"
date: 2023-11-15
author: wsxk
comments: true
---

- [9. 存储器管理（II）：NAND Flash概论　](#9-存储器管理iinand-flash概论)
  - [9.1 Nand Flash 简介](#91-nand-flash-简介)
    - [9.1.1 Nand Flash的特性](#911-nand-flash的特性)
    - [9.1.2 NAND Flash的发展趋势](#912-nand-flash的发展趋势)
    - [9.1.3　如何自动辨认NAND Flash种类？](#913如何自动辨认nand-flash种类)
  - [9.2　控制NAND Flash](#92控制nand-flash)
  - [9.3　Bad Block 管理](#93bad-block-管理)
    - [9.3.1 Initial Bad Block](#931-initial-bad-block)
    - [9.3.2 虚拟/实体位置转换](#932-虚拟实体位置转换)
    - [9.3.3 New Bad Block](#933-new-bad-block)
  - [9.4 ECC（Error Correcting Code）](#94-eccerror-correcting-code)
  - [9.5 平均读写机制](#95-平均读写机制)
    - [9.5.1 Wear-Leveling（耗损平均技术）](#951-wear-leveling耗损平均技术)
  - [9.6 NAAD Flash烧录器：特殊烧录格式](#96-naad-flash烧录器特殊烧录格式)
- [10. 模拟器](#10-模拟器)
- [11. Callback Function](#11-callback-function)
- [12. 用C来实作面向对象的概念 　](#12-用c来实作面向对象的概念-)


## 9. 存储器管理（II）：NAND Flash概论<br>　
`nand flash`:很多电子产品也都以NAND作为大容量存储媒介，但是设计NAND Flash的系统是很大的挑战，原因如下：<br>
```
■　Memory中某些区域的特性是不稳定的（也就是说有些空间是无法使用的）。

■　更麻烦的是每颗IC不稳定区域的数量与位置并不相同。

■　最麻烦的是存储器某些区块在使用后可能会无法写入，甚至数据会丢失。
```
NAND Flash的单位存储器价格相较于ROM、NOR Flash、RAM而言，简直是便宜的不得了！所以业界普遍认为用系统复杂度去取得这个成本优势是值得的<br>

### 9.1 Nand Flash 简介<br>
NAND Flash ‘天生’就有可靠性（Reliability）的问题，这与NAND Flash内部的半导体特性及运行原理息息相关，但我们是做产品与系统开发的，以下我们只会简单说明硬件架构，并将重点摆在如何设计带NAND Flash的系统<br>
首先让我们来看看NAND Flash的硬件架构(2Gb(bit)NAND):<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231119132955.png)
```
■　NAND的基本单位是block，这个设备共有2048个block。
■　每个block里有64个page。
■　每个 page 的 size 是（2048+64 ）Byte，其中 2048 Byte 是 Data Area，而 64 Byte 是 Redundant Area。
```
面上NAND Flash有多种不同的配置，一颗IC里block的数目，一个block里的page数目，每个page的数据区与redundant区的实际大小，都可能随不同厂牌、不同工艺、不同size而不一样，系统必须设计得较具弹性，才有办法支持所有的NAND Flash型号。<br>
此外，不要以为每一个page里的Redundant Area是附送的，Redundant Area是提高NAND Flash Reliability所不可或缺的额外存储区域<br>
**Nand Flash的使用必须面对很多问题**<br>
```
■　Reliability：可靠性。
■　Stronger ECC（Error Correction Code）：ECC是种纠错算法，后面会有专门章节说明。
■　Bad Block管理机制。
■　Optimized Wear-Leveling：平均读写机制，同样在稍后会说明。
■　逻辑与虚拟地址的Mapping：因为每颗NAND Flash的Bad Block分布都不同，所以系统必须将不连续的NAND Flash物理地址（必须跳过Bad Block），包装成连续的虚拟地址。
■　提高性能。
■　新命令：NAND Flash的工艺不断进步，新的NAND Flash可能会提供某厂商或某工艺独有的新命令，而且通常必须使用此命令才能发挥其最好的性能，故系统设计时必须考虑到这一点，系统一定要具备可扩展性。
```
**目前Nand Flash可以分成以下两类：**<br>
```
■　Raw NAND：虽然是目前市场的主流，但规格相当多样且混乱，因为一般中小型厂商不容易取得特定厂商、特定型号的NAND Flash，所以会要求系统必须能够自动判别并支持各种不同属性的NAND Flash
目前市面上的NAND Flash架构有3种：SLC、MLC、TLC。简单地说，MLC是在同一个IC中塞入两个SLC，密度较高、容量较大，单位存储容量较便宜，同时Reliability也较差，而TLC则是塞入3个SLC，单一芯片容量可以做到最大，单位存储容量最便宜、Reliability也最差。
•　SLC：对Reliability要求较高的应用，例如用来取代NOR Flash、用来存储程序代码的应用。•　MLC/TLC：对制造成本较敏感的应用，例如MP3播放器（Apple的iPod、iPhone、iPad就是采用MLC）、SSD(所以ssd是Nand Flash的一种)

■　带Controller的NAND： NAND Flash有着许多已知的问题，对系统开发而言每个都是大麻烦，但采用先进的工艺只能降低单位存储空间的制造成本，并无法解决既有问题（事实上，工艺越进步，NAND Flash的Reliability反而越来越差。
并非所有开发团队都能很好的驾驭NAND Flash这匹野马，所以导致客诉问题层出不穷，造成NAND Flash厂商只能疲于奔命。为此NAND Flash厂商想出在NAND Flash中加入Controller的方法，用内置硬件，处理掉NAND Flash全部或部分的问题，使系统工程师不需再为此烦恼。
几乎所有NAND Flash厂商都推出了这种内置Controller的产品，但系统厂商似乎不愿买单，因为要在NAND Flash内加一颗Controller，显然又会使成本增加。这种NAND Flash分为两类。
□　内置ECC Controller的NAND Flash：例如Hynix的Emulated NAND、Samsung的EF（Error-Free） NAND。
□　内置ECC & Wear-Leveling Controller的NAND Flash：例如Toshiba LBA。
```

#### 9.1.1 Nand Flash的特性<br>
```
■　NAND Flash读出（Read）与写入（Program）的操作是以page为单位，而擦除（Erase）操作则是以block为单位。

■　要先Erase后，才能进行Program，否则数据可能会发生错误（NAND Flash的每个cell在写的时候只能从1→0，不能从0→1；而Erase后，该block中所有的Cell值都是1。所以若要对一个已经使用过的page进行写入的时候，必须先Erase包含这个page的整个block。

■　NOP （Number Of Program）

■　Erase后，每个page能被写入（program）的次数。

■　通常MLC的NOP为1，SLC NOP则不一定。

■　系统为了支持各种NAND Flash，通常会限制每个page只能被program一次。

■　Page Program的顺序

■　SLC可以random page write （page 0，3，1，2 ……）。

■　MLC只能依次往后写（page 0，1，2，3……）。■　系统为了支持各种NAND Flash，通常会限制page program只能依次往后写。

■　NAND Flash中存在Bad Block，且随着对NAND Flash的操作，可能会有新的Bad Block产生，因为对这些Bad Block进行读写的时候，并无法保证数据的完整性，所以系统必须对Bad Block进行相应的处理。

■　相较于其他存储器，NAND Flash的存取速度算是非常慢的: DRAM >> NOR Flash >> NAND Flash >> HDD。

■　常被写的block容易坏，这是由于MLC的特性，在频繁读取数据的时候，里面存储单元的电荷有可能产生变化，从而导致数据错误和遗失。

■　NAND Flash里的任意一个block或page里，随时都可能发生某些bit的值变为错误——如果一个bit坏了就会导致整个page不能使用，那NAND Flash真的可以说是不堪用了！为此，在NAND Flash厂商的‘指导’下，我们导入了ECC的思想。
ECC是个数学算法——若我们说这个是N bit的ECC算法，则代表它最多能修正N个bit的错误

■　常被存取（Read/Program）的page，其附近的page容易坏（称为Read/Write Disturbance）。

■　MLC的部分block具有SLC的特性（Higher Reliability），系统设计可善加应用（但每家每款flash的分布状况并不一定）。建议系统设计者可将‘hot data’写入Reliability较佳的block，至于何谓‘hot data’则视各个系统而定了。”
```

#### 9.1.2 NAND Flash的发展趋势<br>
```
■　更精密的工艺、更大的容量。
■　Block可读写次数减少（从SLC的万次等级，降到今日TLC的小于1000次的等级）。
■　单位售价越来越便宜。
■　必须使用更复杂的error correction algorithm，目前TLC已经需要用到24 bit ECC，其他诸如44 bit ECC、Turbo code、LDPC等算法都已ready，就等着更‘烂’的NAND Flash问世。
■　因为系统与NAND Controller支持新NAND Flash的速度，根本跟不上新NAND Flash对Reliability需求增加的速度，所以NAND Flash厂商仍不会放弃推出带Controller的NAND Flash。
```

#### 9.1.3　如何自动辨认NAND Flash种类？<br>
系统并无法在run-time辨识出板子上NAND Flash的所有属性，在软件不能修改的前提下，只能通过量产工具，在量产期间把目前用以生产之NAND Flash的属性，写入NAND Flash的特定位置，系统才有办法在run-time正确地操作所有的NAND Flash。<br>

### 9.2　控制NAND Flash<br>
**NAND Flash基本操作就是3种：Erase、Program、Read**.驱动程序必须利用‘下命令’的方式，令NAND Flash运作<br>
`Nand flash通常的引脚由如下几种`<br>
```
■　CLE （Command Latch Enable）：下命令时应把此PIN拉为High。
■　CE （Chip Select）：开始操作此NANFD flash时必须把此PIN拉低，有的NAND Flash有多根Chip Select PIN，此时驱动程序要先判断目前要操作的地址属于哪个区块，并控制相应的Chip Select PIN。
■　WE （Write Enable）：在写入数据、命令、地址时，应将此PIN拉Low。
■　ALE （Address Latch Enable）：传输地址时应把此PIN拉为High。
■　RE（Read Enable）：自NAND Flash读取数据时，应把此PIN拉Low。
■　IOx或Data：这8或16根PIN如同Data Bus或Address Bus，NAND驱动程序通过这些PIN传送命令、地址与数据，而NAND Flash则是通过这些PIN传回数据。
■　R/B （Ready/Busy）：当NAND驱动程序把命令下给NAND Flash后，NAND Flash需要时间处理这个命令，它会将R/B PIN拉Low，驱动程序必须等其变回High时才可接续下面的动作（以READ-Command而言，必须等R/B变为High后才能开始读取数据。）。
```
先前提到，nand flash只能按page为单位来写数据，那么修改1字节应该怎么办？<br>
统必须提供单独写入某个page的功能，但NAND Flash的Erase Operation 却是以block为单位，即为了写入某个page，必须Erase一整个block，使得这个block里的其 他page也被‘误杀’。
这是NAND Flash先天的限制，目前解法称为Copy-Back<br>
就是要写入时，找一个空闲block，把修改后的内容从原有的block写入空闲block，然后erase原有block<br>

### 9.3　Bad Block 管理<br>
使用`nand flash`时，有3个机制必不可少<br>
```
■　Bad Block处理
■　ECC（Error Correction Code；错误更正码）
■　Wear-Leveling
```
通常，Bad block包括`刚出厂就已经坏掉的，与使用期间坏掉的`<br>

#### 9.3.1 Initial Bad Block<br>
```
■　NAND Flash出厂时会在Bad Block里的特定位置上标示记号（mark），用来辨别Initial Bad Block。
■　用以标示Initial Bad Block的mark，只要经过Erase后就无法复原。
■　NAND原厂自有其判断某个block是否不堪使用的方式，被标示为initial invalid block的block并非完全无法使用，有的只是在某些状况（例如低电压）比较‘weak’，这也是为什么原厂有时会将其保守的称为‘invalid’block，而不是‘bad’block。但无论叫什么名字，对系统设计者来说这两者都是一样的——就是‘不能用！’
■　接续上一点，系统不能拿Initial Bad Block来用，否则绝对会变成系统里的未爆弹。
■　每种型号的NAND Flash，其Initial Bad Block标示法可能不同，必须参考data sheet。
■　因为Initial Bad Block Information太重要了，而且每颗NAND Flash的Initial Bad Block都分布与数量多寡都不相同，所以系统通常会设法将其备份在NAND Flash的特殊区域里，必要时可取出重建正确的Bad Block Table。
■　接续上一点，在开发阶段，工程师难免会将NAND Flash写烂，此时若Initial Bad Block Information也不见了，则表示这颗NAND Flash已不能用，因为以后系统若发生不稳定的状况，我们很难理清到底是系统的问题，还是Initial Bad Block正在作怪。
```
通常，我们会开发一个小工具，在使用`Nand flash ic`之前，把`Initial Bad Block Information`备份到某个文件里，以后若要使用此NAND Flash，就可将此文件还原到NAND Flash的特殊区域里，以便系统能正确地辨识出Bad Block。<br>

#### 9.3.2 虚拟/实体位置转换<br>
就是很常见的物理地址-》虚拟地址的转换<br>
系统搜集了Initial Bad Block Information后，必须将其记录在一个表格里——`Bad Block Table（以下称之为BB table）`。因为每颗NAND Flash的Bad Block的分布状况与数量都不同，使得系统无法像烧录ROM一样，将image file循序地一个一个Byte写入NAND Flash。所以系统必须基于Bad Block Table，建立一个逻辑位置与实体位置的对应表**这张表就叫做LUT(look-up-table)**。<br>
**物理上nand flash是不连续的，但是逻辑上它是连续的**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231119140608.png)

#### 9.3.3 New Bad Block<br>
系统能得知使用期间block坏掉的情况有以下几种：<br>
```
■　对NAND Flash下命令时，NAND Flash传回Error。
■　从NAND Flash读回的数据，发生ECC Error。
```
为了避免block完全坏掉后才来处理，系统的策略应该是，当发现某block已经有‘快坏掉’的征兆时，就将数据复制到另一个free block，同时修改Look-Up-Table （Logical/Physical Address Mapping Table），并对原来的block执行Erase操作，则此block的特性可恢复，以后还有继续使用的机会。<br>

### 9.4 ECC（Error Correcting Code）<br>
某款NAND Flash里，一个page有（512+16） Byte，系统采用8 bit ECC算法，则当这个page发生错误的bit在8个以下（包含8个）时，这个算法都可以进行修复。**16byte这个关键空间，就是留给ecc算法使用的。当然，这个空间不单单只给ecc使用，还包括了一些文件系统的信息**<br>
ECC算法毕竟不是万能的,如果发生了坏掉的bit比ECC算法能检测或纠正的bit还多的情况，那就完蛋了。<br>
所以系统设计的重点应该是不能任由block烂到极限，应该在快到极限，甚至Error Bit开始陡增之前（例如系统使用8 bit ECC，在发现5～6个Error Bit时），就该将其交换掉，将数据复制到另一个free block，并修改LUT。<br>

### 9.5 平均读写机制<br>
思路很简单，让每个block的使用次数尽量平均，这样就可以延长整个nand flash的使用寿命。<br>
#### 9.5.1 Wear-Leveling（耗损平均技术）<br>
```
■　 Wear-Leveling主要的功能，是让每个block的Program/Erase的次数尽量达到平均。
■　NAND Flash Spec上都会注明每个block可Erase的次数，为了增加Flash的使用寿命，使用Wear-Leveling是完全有必要的。
■　WLC（Wear-Leveling Counter）：系统维护一个表格，称为WLC table，记录每个block到目前为止已经被Erase的次数。
■　系统将整个NAND Flash中没用到的所有block归类到spare区中，如下图所示。
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231119141414.png)
■　Wear-Leveling简单地说，就是系统要maintain一个WLC table，记录每个block被Erase/Program过的次数（称为WLC），若某个block的counter很大，就要去Spare区找一个次数相对少的block来替换，并修改LUT。<br>
■　LUT在Wear-Leveling中扮演着不可或缺的角色，唯有通过修改LUT的mapping，才能做到把WLC较高的block‘置换’为较低的block。<br>

### 9.6 NAAD Flash烧录器：特殊烧录格式<br>
烧录目前广泛用于存储大量数据的NAND Flash可就不是那么简单了，因为NAND Flash工艺与材料的特性，NAND Flash和硬盘一样会有Bad Block，根据原厂规定，刚出厂的NAND Flash芯片的Bad Block数目只要在某个数目以下就不算瑕疵品，更麻烦的是每颗芯片的Bad Block数目与位置都不一样，而且随着操作次数增加，NAND Flash还可能产生新的Bad Block。<br>
当烧录器一个block一个block写入数据时，一旦碰到Bad Block时，势必要找另一个‘Good Block’来代替，而且因为每颗芯片的状况都不一样，最后烧录器一定要把Bad Block的状况记录在NAND Flash之内，那么在执行时期程序才知道该如何跳过Bad Block取得正确的数据。<br>
一般做法是，找到**特定block用作记录用途，■　block 0一定不会是Bad Block**<br><br>
就算程序知道烧录器将BBT存储在block 0中，可是一个block可能是16K Byte，也可能是128KByte，程序怎么知道烧录器将BBT存在block 0的哪里，又怎么知道BBT的格式是什么<br>
**版本答案是FAT**<br>
FAT已普遍应用在PC+Windows平台上的硬盘，硬盘和NAND Flash有许多相同的特性，如操作单位都可以是512 Byte（硬盘称为一个sector,NAND Flash称为page），都会产生新的Bad Block，把FAT应用在NAND Flash上简直就是天作之合。<br>
可以把NAND Flash当成一个磁盘，只要告诉烧录器哪些文件要存到这个‘磁盘’中，烧录器内的程序自然会先对NAND Flash做格式化的动作，然后就像把文件存到随身碟一样，将文件按FAT格式逐一写到NAND Flash中。一但NAND Flash被烧录成FAT格式，就如同机器中多了一颗硬盘一样，所以系统必须支持FAT文件系统才能存取NAND Flash中的数据。<br><br>


## 10. 模拟器<br>
## 11. Callback Function<br>
附录B

## 12. 用C来实作面向对象的概念<br> 　
附录C