---
layout: post
tags: [re]
title: "pyinstaller逆向 & pcapng报文分析 & IDA lumina & AMXX"
date: 2024-2-5
author: wsxk
comments: true
---

- [1. pyinstall re](#1-pyinstall-re)
  - [1.1 曾经的方案](#11-曾经的方案)
  - [1.2 现在的方案pydumpck](#12-现在的方案pydumpck)
- [2. pcapng报文分析](#2-pcapng报文分析)
  - [2.1 安装wireshark](#21-安装wireshark)
  - [2.2 安装pyshark](#22-安装pyshark)
  - [2.3 示例:提取tcp流中的payload](#23-示例提取tcp流中的payload)
- [3. 什么是Lumina](#3-什么是lumina)
  - [3.1 如何使用Lumina](#31-如何使用lumina)
- [4. AMXX re](#4-amxx-re)



## 1. pyinstall re<br>
不知道大家看到这张图片是否熟悉<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240221200006.png)
这张图片常出现在**python代码通过pyinstaller打包成exe可执行程序，exe程序的图标**<br>
如果有逆向的同学们应该就知道这个。<br>
**作为逆向的一个常规套路，当然有常规的解题手段，遇到这种开头的exe文件，直接用pyinstxtractor解包，再用python反编译工具把pyc反编译成py文件**<br>

### 1.1 曾经的方案<br>
**注意，该方案在使用过程中需要确保你正在运行的python版本，和目标exe文件打包时的python版本是一致的（3.x版本一致），反编译出正确值**<br>
如何获取exe文件打包时的python版本？在经过1.1步骤时也能得知。<br>
**pyinstxtractor**<br>
[https://github.com/extremecoders-re/pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor)<br>
下载这个工具后，根据提示运行<br>
```
python pyinstxtractor.py <filename>
```
即可得到解压的目录。<br>
**需要注意的是，pyinstxtractor在解压过程中会列出可能的入口函数的pyc文件，之后反编译优先反编译它们即可**<br>

**uncompyle6反编译**<br>
作为若干年前的神奇，`uncompyle6`给了我很多帮助，然而，`uncompyle6`只支持到`python3.9`版本，在`python3.10`版本后，需要使用其他工具<br>
```
pip install uncompyle6
```
安装完成后，使用<br>
```
uncompyle6 target.pyc > target.py
```
即可<br>

### 1.2 现在的方案pydumpck<br>
[https://github.com/serfend/pydumpck](https://github.com/serfend/pydumpck)<br>
**一键式的解决方案，让反编译变得简单轻松**<br>
```
pip install pydumpck
```
后，使用如下命令即可轻松编译<br>
```
pydumpck xxx.exe
```
得到的结果中含有的`pyc`文件也会自动反编译响应的`python`文件，方便查看~<br>


## 2. pcapng报文分析<br>
最近遇到了关于`misc wireshark`抓包分析的题目，好奇一手做法<br>

### 2.1 安装wireshark<br>
话说回来，`pcapng`报文本身就产生自`wireshark`捕获网络流量报文<br>
所以要想得到`pcapng`报文的内容，还得用`wireshark`启动分析<br>
但是，`wireshark`工具本身虽然提供了友好的用户界面，想要批量分析报文时，还是需要**脚本相助**<br>

### 2.2 安装pyshark<br>
`pyshark`是个比较好用的`python`库，用于处理`wireshark`报文。<br>
```
pip install pyshark
```
**值得注意的是，pyshark默认的tshark.exe(wireshark安装时会存在)的目录在wireshark的默认目录下**<br>
`使用时需要把tshark的文件路径替换成你安装的wireshakr路径`<br>

### 2.3 示例:提取tcp流中的payload<br>
```python
import pyshark

# 指定 pcap 文件路径
file_path = 'your_file_path_here.pcap'

# 创建一个 Capture 对象来读取文件
cap = pyshark.FileCapture(file_path)

# 遍历每个包
for packet in cap:
    # 检查包是否包含 TCP 层
    if 'TCP' in packet:
        # 获取 TCP 层的 payload 属性
        tcp_payload = packet.tcp.payload

        # 去掉 payload 中的冒号
        tcp_payload_clean = tcp_payload.replace(':', '')

        print(tcp_payload_clean)
```
在使用`wireshark`进行分析`TCP`报文时，需要理解**SEQ和ACK的作用：SEQ指当前发送方发送的数据起始位置序列号，ACK指的是当前发送方已经接收到的数据的末尾序列号**<br>
有了这个概念你就比较好分析报文了。<br>
另外，`wireshark追踪流是丢掉报文头部的 只保留了payload`，可以采用原始数据导出.<br>

## 3. 什么是Lumina<br>
Lumina是一种`符号识别服务器`，由IDA专门推出。<br>
Lumina可以在线识别未命名函数<br>

### 3.1 如何使用Lumina<br>
[https://abda.nl/lumen/](https://abda.nl/lumen/)<br>

PS: 跟他类似的还有一个阿里云公开的插件 `Finger`<br>
**注意，在使用Finger时，不要使用科学上网！！！**<br>


## 4. AMXX re<br>
因做到了原题让我感到想吐🤮<br>
[https://in1t.top/2022/06/14/justctf-2022-amxx/](https://in1t.top/2022/06/14/justctf-2022-amxx/)<br>
不过也通过这道题了解了也行`java`和`gradle`的用法，也算不虚此行<br>