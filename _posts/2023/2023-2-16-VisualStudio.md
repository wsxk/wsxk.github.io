---
layout: post
tags: [re]
title: "VS 2022 tips"
date: 2023-2-16
author: wsxk
comments: true
---

- [VS2022 basic knowledge](#vs2022-basic-knowledge)
  - [打开反汇编调试窗口](#打开反汇编调试窗口)
  - [VS命令行工具](#vs命令行工具)
- [gcc编译](#gcc编译)
- [用clang编译](#用clang编译)

## VS2022 basic knowledge<br>
debug: 没有编译优化<br>
release: 编译优化(默认O2)<br>
`其中，O1优化指的是最小文件优化，O2优化指的是最快执行速度优化`<br>

### 打开反汇编调试窗口<br>
调试时使用`f10` 后出现如下窗口<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-1-25-static_analysis_anain/20230216153758.png)
随后按`ctrl+alt+d`出现反汇编窗口<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-1-25-static_analysis_anain/20230216153832.png)
随后可以开始调试

### VS命令行工具<br>
> 1. x64 Native Tools Command Prompt for VS 2022
> 2. x64_86 Cross Tools Command Prompt for VS 2022
> 3. x86 Native Tools Command Prompt for VS 2022
> 4. x86_64 Cross Tools Command Prompt for VS 2022

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-1-25-static_analysis_anain/20230216154059.png)

```
cd /dir
cl /O2 /Fe:name.exe name.c

cl中 /O2指的是优化等级，不添加就默认debug模式
/Fe只程序名称
```

## gcc编译<br>
用`MingW-w64`里的gcc进行编译<br>
```
gcc -m32 -O2 -o xxx.exe xxx.c

其中 -m32 指32位 还有 -m64
-O2指优化
```

## 用clang编译<br>
方法同gcc，只不过 `gcc` 变成了`clang`


