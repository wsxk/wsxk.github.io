---
layout: post
title: "wireshark解密https流量"
tags: [web]
author: wsxk
date: 2022-7-7
comments: true
---

- [1.使用RSA作为密钥交换算法<br>](#1使用rsa作为密钥交换算法)
- [2.使用ECDHE作为密钥交换算法<br>](#2使用ecdhe作为密钥交换算法)
  - [一、设置环境变量<br>](#一设置环境变量)
  - [二、wireshark导入sslkey.log文件<br>](#二wireshark导入sslkeylog文件)
  - [三、开启wireshark，chrome浏览器访问https网站<br>](#三开启wiresharkchrome浏览器访问https网站)

# 1.使用RSA作为密钥交换算法<br>
你需要拥有服务器的私钥（服务器证书对应的私钥），导入到wireshark里。<br>
操作方式，打开wireshark<br>
**edit->reference->protocols->TLS->RSA keys list Edit**
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220707210932.png)
在这里导入rsa的私钥

# 2.使用ECDHE作为密钥交换算法<br>
使用ECDHE算法作为密钥交换算法时，即使拥有rsa私钥也无法解密流量。<br>
考虑使用chrome的SSLKEYLOGFILE接口<br>
## 一、设置环境变量<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220707211130.png)
其中 变量名不能修改，变量值（即路径）你可以随意选择你电脑的任意目录。<br>
在你选择的目录下创建 sslkey.log文件

## 二、wireshark导入sslkey.log文件<br>
**edit->reference->protocols->TLS->(Pre)-Master-Secret log file name**
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220707211309.png)

## 三、开启wireshark，chrome浏览器访问https网站<br>
这样你的wireshark就会截获并自动解析https流量包。<br>

