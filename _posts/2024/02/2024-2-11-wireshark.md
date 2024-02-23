---
layout: post
tags: [web]
title: "wireshark pcapng报文分析"
date: 2024-2-11
author: wsxk
comments: true
---

- [前言](#前言)
- [安装pyshark](#安装pyshark)
- [示例:提取tcp流中的payload](#示例提取tcp流中的payload)


## 前言<br>
最近遇到了关于`misc wireshark`抓包分析的题目，好奇一手做法<br>

## 安装pyshark<br>
`pyshark`是个比较好用的`python`库，用于处理`wireshark`报文。<br>
```
pip install pyshark
```
**值得注意的是，pyshark默认的tshark.exe(wireshark安装时会存在)的目录在wireshark的默认目录下**<br>
`使用时需要把tshark的文件路径替换成你安装的wireshakr路径`<br>

## 示例:提取tcp流中的payload<br>
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