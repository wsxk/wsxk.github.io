---
layout: post
title: "VScode ssh remote办公"
date:   2022-5-23
tags: [software_build]
comments: true
author: wsxk
---

- [1.安装VScode插件 remote_development](#1安装vscode插件-remote_development)
- [2.创建ssl_key](#2创建ssl_key)
- [3.虚拟机安装ssh服务](#3虚拟机安装ssh服务)
- [4.移动id_rsa.pub到虚拟机](#4移动id_rsapub到虚拟机)
- [5.连接](#5连接)

不知道米娜桑有没有遇到这样一个问题

你一些程序要在linux上跑，可是你的物理机是windows，你要在虚拟机上写程序然后运行它

其实按理来说也没什么，但是我最近在写代码发现，在虚拟机上写代码，不好分屏（不能边看帮助边写代码）

然后发现VScode有 remote ssh可以帮助我们在物理机上编写linux虚拟机里的代码。十分的方便，于是搞一搞

## 1.安装VScode插件 remote_development

你需要在你的物理机VScode上安装插件（其实这个版本只需要remote-SSH就行了）

但是你一定要装 remote development插件！！ 没有它你不但不能调试，你还看不到补全，十分难受！！！

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-23-VScode_remote/6.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-23-VScode_remote/1.png)


## 2.创建ssl_key

你需要在你的物理机上运行（我的物理机是win11，自带了，现在是个机子都会带ssh吧）

    ssh-keygen -t rsa -b 4096

运行完会在

    C:\Users\your_username\.ssh的目录下发现2个文件

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-23-VScode_remote/2.png)

## 3.虚拟机安装ssh服务

我们的虚拟机必须带有ssh服务才行（我的虚拟机是ubuntu16 于是）

    sudo apt-get install openssh-server
    sudo service ssh start

## 4.移动id_rsa.pub到虚拟机

用刚刚在第二步生成的 id_rsa.pub复制到虚拟机的

    ~/.ssh 目录下

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-23-VScode_remote/3.png)

## 5.连接

在物理机的VSCODE上运行

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-23-VScode_remote/4.png)

然后添加一个新的用户

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-23-VScode_remote/5.png)

user指的是你在虚拟机里的用户名称

server_ip就是虚拟机的ip地址

接下来就很简单，输入密码（连上了会让你输你的用户名对应的密码）

