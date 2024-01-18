---
layout: post
tags: [software_build]
title: "docker ubuntu install"
date: 2023-2-16
author: wsxk
comments: true
---

- [安装docker](#安装docker)
- [普通用户运行docker](#普通用户运行docker)
- [references](#references)


## 安装docker<br>
[https://docs.docker.com/engine/install/ubuntu/#set-up-the-repository](https://docs.docker.com/engine/install/ubuntu/#set-up-the-repository)

## 普通用户运行docker<br>
```
sudo groupadd docker  //创建docker用户组
sudo gpasswd -a ${USER} docker //将当前用户加入docker用户组
sudo service docker restart //重启docker 服务
sudo chmod a+rw /var/run/docker.sock //为user增加权限
```
## references<br>
[https://www.muzhuangnet.com/show/79356.html](https://www.muzhuangnet.com/show/79356.html)<br>
