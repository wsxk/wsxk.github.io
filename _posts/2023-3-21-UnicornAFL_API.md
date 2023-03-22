---
layout: post
tags: [iot]
title: "UnicornAFL Harness"
author: wsxk
date: 2023-3-21
comments: true
---


- [Unicorn](#unicorn)
  - [Unicorn VS qemu](#unicorn-vs-qemu)
- [UnicornAFL install](#unicornafl-install)
- [Unicorn API](#unicorn-api)
- [实践](#实践)


## Unicorn<br>
基于qemu的另一个开源项目。<br>
现在变成了基础设施之一。像 angr，radare2都集成了Unicorn框架<br>
### Unicorn VS qemu<br>
总得来说，unicorn专注于CPU的指令，不提供qemu那样对计算机其他设备的模拟。<br>
优势还是有很多的:<br>

> 1. 框架：容易集成
> 2. 灵活：即使没有上下文信息，即便没有文件格式（ELF），也能模拟运行，使用于dump出来的二进制片段
> 3. 插桩： 提供qemu没有提供的动态插桩，能够自定义插桩技术
> 4. 线程安全： qemu一般只能运行一个cpu，unicorn可以同时运行多个
> 5. bindings： 说白了就是提供了各种语言的API接口，方便开发
> 6. 轻量： 看大小就明白了，你安装qemu要装半天，unicorn很快就搞定了
> 7. 安全： 因为模拟的方面少（就cpu），所以攻击面小（😀


## UnicornAFL install<br>
首先download`AFL++`,网址在[https://github.com/AFLplusplus/AFLplusplus](https://github.com/AFLplusplus/AFLplusplus)<br>
然后去unicorn_mode目录下运行`./build_unicorn_support.sh`<br>
[https://github.com/AFLplusplus/AFLplusplus/tree/stable/unicorn_mode](https://github.com/AFLplusplus/AFLplusplus/tree/stable/unicorn_mode)

## Unicorn API<br>
有一些API需要了解<br>
网址在这里[https://github.com/kabeor/Unicorn-Engine-Documentation/blob/master/Unicorn-Engine%20Documentation.md#uc_mem_write](https://github.com/kabeor/Unicorn-Engine-Documentation/blob/master/Unicorn-Engine%20Documentation.md#uc_mem_write)<br>
自行翻阅


## 实践<br>
写了个程序
