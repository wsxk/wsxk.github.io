---
layout: post
title: "hashcat GPU bruteforce"
tags: [crypto]
date: 2024-12-31
author: wsxk
comments: true
---

- [0. 写在前面](#0-写在前面)
- [1. gpu components install](#1-gpu-components-install)
  - [1.1 显卡](#11-显卡)
  - [1.2 显卡驱动](#12-显卡驱动)
  - [1.3 cuda toolkits](#13-cuda-toolkits)
- [2. hashcat bruteforce](#2-hashcat-bruteforce)
  - [2.1 hashcat install](#21-hashcat-install)
  - [2.2 hashcat 用法](#22-hashcat-用法)
    - [2.2.1 爆破sha256](#221-爆破sha256)


# 0. 写在前面<br>
在进行渗透测试工作时，我发现在某个场景下，需要了解当前硬件能力在爆破8字节密码的耗时。<br>
这不正巧买了块4070tis，用它来试试爆破密码的能力。<br>

# 1. gpu components install<br>
首先搞定显卡的部分<br>
在装显卡时，我们需要明确一下概念:<br>
```
1、 显卡：实际的物理设备
2、 显卡驱动： 让操作系统能够与显卡进行交互的桥梁
3、 cuda toolkits： 类似于一系列c语言的库和编译器，这里指的是通过编程的方法来操控显卡的一系列库、工具以及编译器(指的是nvcc)
```

## 1.1 显卡<br>
物理设备就需要靠你自己买了，我买的是4070tis，你买什么你自己看着办（😀<br>
买了就装上去，我这里遇到一个坑是机器识别不到显卡。这个问题我询问客服后的解决办法是关闭电源，重新拔插显卡相关的电源线，重启后就解决了<br>

## 1.2 显卡驱动<br>
请到invidia官网下载显卡驱动[https://www.nvidia.cn/geforce/drivers/](https://www.nvidia.cn/geforce/drivers/)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241225210039.png)
一般会有两种类型的驱动:`GeForce Game Ready 驱动程序 - WHQL和 NVIDIA Studio 驱动程序 - WHQL`<br>
如果还要兼顾玩游戏建议选前者<br>

## 1.3 cuda toolkits<br>
装完显卡驱动后，可以通过命令行执行`nvidia-smi`来查看当前的`cuda driver`版本<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241225210029.png)
注意：**cuda driver版本是后向兼容(向过去兼容)的，这意味着我们安装的cuda toolkits只要版本 小于等于12.7 都是可以用的**<br>
接下来我们需要去英伟达官网下载`cuda toolkits`<br>
[https://developer.nvidia.com/cuda-downloads](https://developer.nvidia.com/cuda-downloads)<br>
下载并执行`cuda toolkits`时，安装程序会提醒你先安装`visual studio`，这边也建议你提前安装好<br>
安装完`visual studio`再安装`cuda toolkits`后，在命令行尝试输入`nvcc --version`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241225210441.png)
安装成功<br>



# 2. hashcat bruteforce<br>
## 2.1 hashcat install<br>
前往官网[https://hashcat.net/hashcat/](https://hashcat.net/hashcat/)下载即插即用的hashcat程序<br>
为确保先前的工作都成功了，可以在命令行运行`.\hashcat.exe -I`,如果出现以下情况，说明安装成功:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241225210712.png)

## 2.2 hashcat 用法<br>
接下来就到了hashcat的用法教学了！<br>
首先，可以运行`./hashcat.exe --help`来查看所有参数相关的教学:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241225212807.png)

### 2.2.1 爆破sha256<br>
```python
from hashlib import sha256

password = b"123456"
password_sha256 = sha256(password).hexdigest()
print("sha256:",password_sha256)
# sha256: 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92
```
生成了sha256后，执行如下命令:<br>

``` 
.\hashcat.exe -a 3 -w 3 -m 1400 8d969eef6ecad3c29a3a629280e686cf0c3f5d5a86aff3ca12020c923adc6c92 ?a?a?a?a?a?a

-a 3 表示暴力破解
-w 3 启用特定的工作负载配置文件
-m 1400 爆破的模式，1400说明爆破sha256
hash
?a?a?a?a?a?a: 表示爆破的密码的信息，这里就表示6个字节的可见字符
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20241225213147.png)
爆破成功，GPU爆破比我想象的还牛<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>