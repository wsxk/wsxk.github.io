---
layout: post
tags: [iot]
title: "halfuzz"
date: 2023-4-1
author: wsxk
comments: true
---

- [install](#install)
  - [安装python](#安装python)
  - [安装python3虚拟环境](#安装python3虚拟环境)
  - [使用python3虚拟环境进行安装](#使用python3虚拟环境进行安装)
- [安装hal\_fuzz](#安装hal_fuzz)
- [references](#references)


## install<br>
**环境 ubuntu18.04**<br>
### 安装python<br>

    sudo apt install python
    sudo apt install python-pip
    sudo apt install python3
    sudo apt install python3-pip

### 安装python3虚拟环境<br>

    sudo pip3 install virtualenv
    sudo pip3 install virtualenvwrapper

在普通用户下，进入

    gedit ~/.bashrc

在末尾加入内容

    export WORKON_HOME=$HOME/.virtualenvs
    export VIRTUALENVWRAPPER_PYTHON='/usr/bin/python3.6'
    source /usr/local/bin/virtualenvwrapper.sh

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230401132130.png)

### 使用python3虚拟环境进行安装<br>

首先从github上clone hal-fuzz下来，然后

    python3 -m venv hal-fuzz-python
    source hal-fuzz-python/bin/activate
    sudo ./setup.sh
    
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230401132130.png)
保存后推出，使用如下命令

    source ~/.bashrc

接下来就可以使用 mkvirtualenv来创建虚拟环境<br>

## 安装hal_fuzz<br>
官网在这里[https://github.com/ucsb-seclab/hal-fuzz](https://github.com/ucsb-seclab/hal-fuzz)<br>
根据这个命令做就好了

## references<br>
[https://www.dgrt.cn/news/show-128188.html?action=onClick](https://www.dgrt.cn/news/show-128188.html?action=onClick)<br>