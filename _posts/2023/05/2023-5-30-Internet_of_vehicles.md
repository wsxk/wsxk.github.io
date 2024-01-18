---
layout: post
tags: [iot]
title: "Internet of vehicles learning"
date: 2023-5-31
author: wsxk
comments: true
---

PS: 更新于`2023-5-31`<br>

- [前言](#前言)
- [1. CAN总线](#1-can总线)
  - [一、高速CAN总线](#一高速can总线)
  - [二、低速CAN总线](#二低速can总线)
  - [三、CAN总线标准数据帧](#三can总线标准数据帧)
- [2. 统一诊断服务(uds)](#2-统一诊断服务uds)


## 前言<br>
高贵的上饶杯车联网安全挑战赛，我表示十分地神秘.<br>
虽然日车不能带自己的设备，全程要有人跟着看...anyway，还是多少学了点 车联网的内容<br>
`PS:有关车联网的内容，之后如果还学到了，会继续更新`<br>

## 1. CAN总线<br>
控制器局域网总线（CAN，Controller Area Network）是一种用于实时应用的串行通讯协议总线，它可以使用双绞线来传输信号，是世界上应用最广泛的现场总线之一。 CAN协议用于汽车中各种不同元件之间的通信，以此取代昂贵而笨重的配电线束。 该协议的健壮性使其用途延伸到其他自动化和工业应用。<br>
**CAN总线其实相当于物理层和数据链路层**<br>
### 一、高速CAN总线<br>
高速CAN总线最长可达40m，速率最大可达1M/s<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230602112037.png)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230602112354.png)

### 二、低速CAN总线<br>
低速CAN总线最长可达1km，速率最大125kb/s<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230602112207.png)

### 三、CAN总线标准数据帧<br>
**3字节信息帧和8字节数据帧**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230602112516.png)

## 2. 统一诊断服务(uds)<br>
UDS（Unified Diagnostic Services）协议是一种被广泛应用在汽车行业的通信协议。它是一种标准化的诊断通信协议，用于在车辆的电子控制单元（ECU）之间传输信息。这种协议通常运行在CAN（Controller Area Network）总线上，但也可以使用其他类型的网络<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230602112748.png)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230602112946.png)
从右侧的数据报文中可以看到 `27 01`是对编号为`01`的设备进行诊断，`67 01`是相应的回复报文<br>
**统一诊断服务相当于应用层协议**<br>