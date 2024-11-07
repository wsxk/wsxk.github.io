---
layout: post
title: "ubuntu18 llvm install"
tags: [software_build]
date: 2023-3-1
author: wsxk
comments: true
---

- [下载之前](#下载之前)
- [1.下载源码](#1下载源码)
- [2. 安装编译工具链](#2-安装编译工具链)
- [3. 安装](#3-安装)


llvm安装还是很麻烦的（各个超大型的应用程序都这个样子<br>

## 下载之前<br>
之前开了ubuntu18 的虚拟机 4g内存也炸了，开了8g内存后成功编译运行。<br>
建议尽可能的开大内存，否则你会遭遇玄学问题<br>

## 1.下载源码

首先需要去github下载源码<br>
```shell
git clone https://github.com/llvm/llvm-project
```
如果没有git 请`apt install git`<br>

## 2. 安装编译工具链<br>
```shell
apt install make
apt install cmake
apt insatll gcc
apt install g++
```

## 3. 安装<br>
```shell
cd llvm-project
git checkout release/10.x
mkdir build
cd build

cmake -G "Unix Makefiles" -DCMAKE_BUILD_TYPE=Release --enable-optimized --enable-targets=host-only  ../llvm -DLLVM_ENABLE_PROJECTS="clang;libcxx;libcxxabi;compiler-rt;clang-tools-extra;openmp;lldb;lld" 

make -j4
make install
```
可能还会遇到一些错误，基本上是缺少什么库的，根据报错安装即可。<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>