---
layout: post
title: "win11 MobSF构建过程"
date:   2022-1-27
tags: [software_build]
comments: true
author: wsxk
---

MobSF是一款移动应用分析框架，可以做到android、iOS应用的静态和动态分析

目录
- [安装过程](#安装过程)
  - [1.准备工作](#1准备工作)
    - [(1)git](#1git)
    - [(2)python3.8-3.9](#2python38-39)
    - [(3)JDK 8+](#3jdk-8)
    - [(4)Microsoft Visual C++ Build Tools](#4microsoft-visual-c-build-tools)
    - [(5)OpenSSL (non-light)](#5openssl-non-light)
    - [(6)wkhtmltopdf](#6wkhtmltopdf)
  - [2.下载源码](#2下载源码)
  - [3.测试](#3测试)


### 安装过程
#### 1.准备工作
MobSF安装需要以下依赖环境
(1)git
(2)python3.8-3.9
(3)JDK 8+
(4)Microsoft Visual C++ Build Tools
(5)OpenSSL (non-light)
(6)wkhtmltopdf 

##### (1)git
git 安装比较简单，网上安装的博客也很多，跟着其中一篇安装即可。

##### (2)python3.8-3.9
python 安装也比较简单，注意要是python3.8-3.9之间的任意版本（包括3.8和3.9）

##### (3)JDK 8+
去java官网选最新的下即可（任意8以上版本JDK均可）
这里注意，在输入环境变量时一定要添加JAVA_HOME变量！！！！！
不然在实际安装必定出错。
JAVA_HOME变量填的是你安装java的目录
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-27-MobSF_create1.png)

在系统变量中的“Path”中填入
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-27-MobSF_create2.png)

完成上述步骤后，测试是否java环境变量是否添加完毕
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-27-MobSF_create3.png)

##### (4)Microsoft Visual C++ Build Tools
链接贴在下面
[网址](https://visualstudio.microsoft.com/zh-hans/thank-you-downloading-visual-studio/?sku=BuildTools&rel=16)
直接运行安装即可（不用任何改动，运行exe后直接选择安装选项）

##### (5)OpenSSL (non-light)
链接贴在下面
[网址](https://slproweb.com/products/Win32OpenSSL.html)
注意，选择的版本一定不是light版的！！！

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-27-MobSF_create4.png)
安装这个安装默认安装！！！一定要安装默认安装！！！跟着默认选项走运行时不会出错！！！

##### (6)wkhtmltopdf 
链接贴在下面
[网址](https://wkhtmltopdf.org/downloads.html)
安装完成后注意，要把安装目录下的bin文件夹添加到系统环境变量中！

#### 2.下载源码
敲下面命令即可

    git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF.git
    cd Mobile-Security-Framework-MobSF
    setup.bat

#### 3.测试

在安装完成后运行

    ./run.bat 127.0.0.1:8000

在浏览器中输入这个网址

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-27-MobSF_create5.png)

大功告成