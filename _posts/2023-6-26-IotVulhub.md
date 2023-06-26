---
layout: post
tags: [iot]
title: "IoT-vulhub 漏洞复现"
date: 2023-6-26
author: wsxk
comments: true
---

- [写在前面](#写在前面)
- [install](#install)
- [1. Vivotek CC8160](#1-vivotek-cc8160)
  - [install](#install-1)


## 写在前面<br>
~~作为一名物联网安全研究员，怎么能不复现一下漏洞呢~~<br>
然而，为了一个漏洞去买一个设备感觉还是有点那啥（倒不是说没必要，新手可能需要先试试手，不至于浪费💴）<br>
`IoT-vulhub`就是一个有用的物联网漏洞复现平台，其集成了`firmadyne`以及`binwalk`，十分适合练手~<br>

## install<br>
建议使用`ubuntu20.04`进行安装，因为其他环境我也没装过~<br>
**第一步，安装pip,建议在root下安装**<br>
```shell
curl -s https://bootstrap.pypa.io/get-pip.py | python3
```
**第二步，安装docker**<br>
```shell
curl -s https://get.docker.com/ | sh
```
顺带一提，为了能够让用户态进行docker访问，你需要参照[https://wsxk.github.io/docker_install/](https://wsxk.github.io/docker_install/)的步骤进行。<br>
**第三步，启动docker环境以及安装docker-compose**<br>
```shell
systemctl start docker
python3 -m pip install docker-compose
```
值得注意的是，缺什么就安装什么，所有报错均可以百度搜索到~<br>
安装完环境后，即可开始漏洞复现工作。<br>
## 1. Vivotek CC8160<br>
### install<br>
跟随[https://github.com/VulnTotal-Team/IoT-vulhub/tree/master/VIVOTEK/remote_stack_overflow](https://github.com/VulnTotal-Team/IoT-vulhub/tree/master/VIVOTEK/remote_stack_overflow)搭建环境即可~我才用的是系统模拟的方法<br>
顺道一提，在根据步骤操作到`构建镜像`时会出现问题，具体原因是`docker.io`提供的网站并没有`firmianay/qemu-system:armel`这个镜像，因此你需要自己在本地安装。<br>
**IoT-vulhub-master/baseImage/qemu-system/armel目录下，有一个download.sh文件，把其中的下载链接全都改成https://file.erlkonig.tech/debian-armel/xxxx 即可**<br>
随后使用 `docker build -t firmianay/qemu-system:armel .`在本地构建镜像即可。<br>
还是在`构建镜像`步骤，在新版本docker中，执行`system-emu`目录下的`dockerfile`时会出现问题。`COPY ./firmware/_*/_31* /root/firmware`会执行失败。**因为在新版docker中，会对文件名做检测，所以，建议在文件中找到你提取的带有_31的目录到本地下，同时将命令改成`COPY ./firmware/_31.extracted /root/firmware`即可。**<br>

