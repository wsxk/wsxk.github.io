---
layout: post
tags: [software_build]
title: "ubuntu18 安装多个版本python3&pip3"
date: 2023-3-27
author: wsxk
comments: true
---


在做实验的时候安装东西碰到了神秘问题（...有的代码在python3.6版本上会跑出问题，而在python3.7的版本上能正常使用，这就涉及到了同时存在多个python版本的问题上。<br>
所以来记录一下ubuntu18的安装多个python3和pip3版本的步骤。<br>

## 前期准备<br>

    sudo apt update
    sudo apt install curl
    apt install python3.6
    apt install python3.7


## 下载pip3安装脚本<br>

    curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
    python3.6 get-pip.py
    python3.7 get-pip.py

搞定