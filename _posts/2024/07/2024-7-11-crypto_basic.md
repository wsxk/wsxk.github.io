---
title: "Cryptography"
tags: [crypto]
layout: [post]
date: 2024-7-11
author: wsxk
comments: true
---

- [前言](#前言)
- [1. 密码学三要素](#1-密码学三要素)
- [2. One-Time Pad (OTP)](#2-one-time-pad-otp)


## 前言<br>
常见的密码算法编写可看[Re 常见加解密算法识别与加解密脚本](https://wsxk.github.io/ctf_common_re/)<br>
（又是一个自导自演~<br>

## 1. 密码学三要素<br>
这是一个~<br>
```
1. Confidentiality（机密性）
就是不希望A和B的通信内容被第三方C看到~

2. integrity（完整性）
就说不希望A和B的通信内容被第三方篡改~

3. Authenticity(真实性)
就是希望A真的是和B在通话，而不是装成B的C
```
简单的加解密逻辑:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240711213949.png)
加密： 明文在key和加密函数的作用下得到密文。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240711214006.png)
解密： 密文在key和解密函数的作用下得到明文<br>

## 2. One-Time Pad (OTP)<br>
`OTP,别称一次性密码本`,是很经典的古典加密方法之一。<br>

