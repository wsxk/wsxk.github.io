---
layout: post
tags: [iot]
title: "qemu instructions"
date: 2023-3-12
author: wsxk
comments: true
---

- [qemu-system-arm](#qemu-system-arm)


## qemu-system-arm<br>

    ./qemu-system-arm -M help #查看可支持的架构
    
    ./qemu-system-arm -M netduinoplus2 -cpu cortex-m4 -m 1M -nongraphic -d in_asm.nochain -kernel task1.elf -D log.txt
    # -M选择开发板架构 
    #-cpu选择处理器架构 
    #-m 内存大小
    #-nongraphic无图形化界面 
    #-d 调试日志输出文件 
    #-kernel 固件 
    #-D 选择输出日志