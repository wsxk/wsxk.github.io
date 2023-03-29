---
layout: post
tags: [software_build]
title: "阿里云ecs构建Intel SGX环境"
date: 2023-3-29
author: wsxk
comments: true
---


- [前言](#前言)
- [prerequisites](#prerequisites)
  - [how to check](#how-to-check)
- [为什么用阿里云ecs](#为什么用阿里云ecs)
  - [how to use](#how-to-use)


## 前言<br>
7th XCTF总决赛时遇到一个pwn题目，是关于Intel SGX的，没有做出来（配环境配了大半天，乐<br>

## prerequisites<br>
首先，你的设备CPU必须是intel的<br>
其次，你的CPU必须支持Software Guard Extensions (SGX)。<br>

### how to check<br> 
值得一提的是，因为题目环境使用的是`ubuntu18.04`<br>
所以可以使用`cpuid | grep -i sgx`来检测是否硬件是否支持。<br>
如果支持，应该可以看到如下实例<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230329235756.png)

## 为什么用阿里云ecs<br>
其实是因为本机上使用的vmware不支持 `SGX`,需要安装特定版本的vmware，比如 `VMware vSphere 6.7 U3及更高版本`。但是我不想在本机上折腾（其实硬盘容量也不够了）。<br>
于是乎想着能不能在云服务器上搞一下。<br>

### how to use<br>
首先要说的是，**轻量应用服务器是不支持`SGX`的**，<br>
请务必注意。<br>
我们应该选择的是 阿里云的云服务器ECS(Elastic Compute Service)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230330000350.png)

选择立即购买后，跳过新人免费试用，直接购买（别问，新人试用不让你玩SGX）<br>
选型如下:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230330000714.png)
选**c7t、g7t**都可以<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230330000748.png)

然后镜像选择`ubuntu18.04 uefi版`可以直接下一步了。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230330000837.png)

接下来就没有什么难点，一路点击继续，交钱就完事了。<br>
然后就可以用ssh连接进服务器实例。<br>