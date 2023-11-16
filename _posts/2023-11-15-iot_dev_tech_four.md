---
layout: post
tags: [iot_dev]
title: "iot dev technology 4：NAND Flash & 模拟器 & call back"
date: 2023-11-15
author: wsxk
comments: true
---

- [9. 存储器管理（II）：NAND Flash概论　](#9-存储器管理iinand-flash概论)
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


## 10. 模拟器<br>
## 11. Callback Function<br>
附录B

## 12. 用C来实作面向对象的概念<br> 　
附录C