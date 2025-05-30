---
layout: post
tags: [re]
title: "Assembly Instruction"
date: 2023-5-2
author: wsxk
comments: true
---

- [写在前面](#写在前面)
- [1. x86\_64](#1-x86_64)
  - [浮点运算](#浮点运算)
  - [跳转指令合集](#跳转指令合集)
- [2. arm32](#2-arm32)
  - [neg](#neg)
  - [IT{x{y{z}}} {cond}](#itxyz-cond)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## 写在前面<br>
本章用来记录一下不常见的，各种架构下的汇编指令。<br>

## 1. x86_64<br>
### 浮点运算<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230503130250.png)
### 跳转指令合集<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230504191152.png)


## 2. arm32<br>
### neg<br>
其实是求补码的指令,即取反后加1

1111（-1） 求补码为 0001（1）

### IT{x{y{z}}} {cond}<br>
```
IT (If-Then) 指令由四条后续条件指令（IT 块）句组成。 这些条件可以完全相同，也可以互为逻辑反。
IT 块中的指令（包括跳转）还必须在语法的 {cond} 部分中指定条件。
无需在代码中编写 IT 指令，因为汇编器会根据在后续指令中指定的条件为您自动生成这些指令。 不过，如果确实   需要编写 IT 指令，则汇编器会根据后续指令中指定的条件对 IT 中指定的条件进行验证。
编写 IT 指令可确保您会考虑如何在代码设计中放置条件指令以及选择条件。
在汇编为 ARM 代码时，汇编器会执行相同的检查，但是不会生成任何 IT 指令。


限制
不允许在 IT 块中使用下面的指令：
IT
CBZ 和 CBNZ
TBB 和 TBH
CPS、CPSID 和 CPSIE
SETEND
使用 IT 块时的其他限制有：
跳转指令或修改 pc 的任何指令只能是 IT 块中的最后一个指令。
无法跳转到 IT 块中的任何指令，除非在从异常处理程序返回时。
不能在 IT 块中使用任何汇编器指令
``` 
**本质上是一个条件判断指令，判断接下来的1~4个指令是否可以执行**
> IT{x{y{z}}} {cond}
> x表示第二条指令的条件开关，y代表第三条指令的条件开关，z表示第四条指令条件开关
> x，y，z可以是 **T（then）或E（else）** 中的一种
> T 表示在满足cond时执行，E表示在不满足cond时执行


可以看看[这篇文章](https://juejin.cn/post/6844903968972210190)
