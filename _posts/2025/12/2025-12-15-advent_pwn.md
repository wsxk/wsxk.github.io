---
layout: post
tags: [ctf_wp]
title: "advent of pwn 2025"
author: wsxk
date: 2025-12-15
comments: true
---

- [day 1: check-list](#day-1-check-list)
- [day 2: coal](#day-2-coal)
- [day 3: ](#day-3-)


# day 1: check-list<br>
超长汇编构成一个函数执行（ida无法反编译），函数读取256字节输入，经过变换后，得到的输出与结果比较，正确就输出flag<br>
**经过分析256个字节的变换是以一个字节为单位的（其字节A和字节B的变换没有相关性）的线性函数(只有add和sub两个汇编指令)：yi = xi + bi**<br>
解题思路是： <br>
```
1. 输入全为0，得到256个线性函数 yi = xi +bi 中的bi数组
2. ida脚本提取 yi
3. xi = yi-bi
```
得到正确输入。<br>


# day 2: coal<br>
根据题目的意思：可能某些命令的组合条件能够**dump**出内存，即普通用户能够dump出root用户进程的内存。<br>
相关环境的命令:<br>
```
nice(1), core(5), elf(5), pty(7), signal(7)
```
设置`ulimit -c unlimited`后，执行程序然后`ctrl+\`dump出内存即可。<br>
切换到练习模式获得root，改文件权限然后看内存即可~<br>

# day 3: <br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>