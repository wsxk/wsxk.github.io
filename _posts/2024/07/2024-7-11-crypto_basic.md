---
layout: post
title: "Cryptography"
tags: [crypto]
date: 2024-7-11
author: wsxk
comments: true
---

- [前言](#前言)
- [1. 密码学三要素](#1-密码学三要素)
- [2. One-Time Pad (OTP)](#2-one-time-pad-otp)
- [3. symmetric encrypt(对称加密)](#3-symmetric-encrypt对称加密)
  - [3.1 Encryption Properties](#31-encryption-properties)
  - [3.2 Advanced Encryption Standard(AES)](#32-advanced-encryption-standardaes)
    - [3.2.1 aes padding: Padding with Null Bytes](#321-aes-padding-padding-with-null-bytes)
    - [3.2.2 aes padding: Padding with PKC#7](#322-aes-padding-padding-with-pkc7)
  - [3.3 AES加密模式](#33-aes加密模式)
    - [3.3.1 ECB(Electronic Codebook)](#331-ecbelectronic-codebook)
    - [3.3.2 cbc(Cipher Blocking Chaining)](#332-cbccipher-blocking-chaining)
    - [3.3.3 CTR(Counter)](#333-ctrcounter)

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
它的加解密方法如下：<br>
```
加密方式：
1. Randomly generate a key: 随机生成一个key
    其中key的每一个bit为0/1的概率相等
2. 把明文plaintext与key的每一个bit进行xor操作，得到密文ciphertext
    这能保证明文中每个比特被翻转的概率相等

解密方式：
把ciphertext的每个比特与key的每个比特进行xor操作，就能得到明文plaintext
    因为xor的操作是可逆的：
    (a ⊕ b) ⊕ b = a ⊕ (b ⊕ b) = a ⊕ 0 = a
```


## 3. symmetric encrypt(对称加密)<br>
在提到对称加密前，不得不提到**混淆（confusion）与扩散（diffusion**这两个操作了<br>
### 3.1 Encryption Properties<br>
混淆和扩散是当今密码学中，强加密算法的核心操作。<br>
```
confusion:混淆
使密文的每个比特取决于密钥的几个部分，是一种使密钥与密文之间的关系尽可能模糊的加密操作。
如今实现混淆常用的一个元素就是替换；这个元素在 AES 和 DES 中都有使用。

diffusion：扩散
明文的1bit的改变可以导致整个密文有大约一半的比特位被改变。是一种为了隐藏明文的统计特性而将一个明文符号的影响扩散到多个密文符号的加密操作。
最简单的扩散元素就是位置换，它常用于 DES 中；而 AES 则使用更高级的 Mixcolumn 操作。
```

### 3.2 Advanced Encryption Standard(AES)<br>
AES加密就是很常见的对称加密方式了。<br>
加密：plaintext在 key和aes加密的作用下生成 ciphertext<br>
解密：ciphertext在 key和aes解密的作用下生成 plaintext<br>
但是有一点要注意，对于aes加密来说,key的长度和plaintext的每个block长度是有要求的:<br>
```
Key Size: (128/192/256)-bits
Block Size: 128-bits
```
因此，对于明文为`Hello, World!`，长度是不足以用来加密的，所以需要`padding`，所谓`padding`，就是在plaintext的末尾添加几个字符，使plaintext能够满足加密的最小长度需要<br>
#### 3.2.1 aes padding: Padding with Null Bytes<br>
顾名思义，就是在plaintext后面填充`0`字节<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240712191349.png)

#### 3.2.2 aes padding: Padding with PKC#7<br>
这个`padding`的方式就是在末尾补齐字符，字符的值为要补齐的字节长度。<br>
比如`Hello, World!`,长度为13，需要再补3个字节，所以值就是`\x03\x03\x03`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240712191725.png)
对于``Hello, World!Hello, World!`,长度为26，需要再补6个字节，所以值就是`\x06\x06\x06\x06\x06\x06`<br>

### 3.3 AES加密模式<br>
之前提供，aes加密每次只能加密plaintext的16个字节（一个block），所以根据每个block的加密的关联关系，划分了不同的加密模式<br>

#### 3.3.1 ECB(Electronic Codebook)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240712192458.png)
好处：ecb模式的每个block的加密都是独立的，可以**并行运算**<br>
坏处：如果plaintext是有规律的，规律也会体现在ciphertext上<br>

#### 3.3.2 cbc(Cipher Blocking Chaining)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240712192636.png)
好处：plaintext的规律不会体现在ciphertext上<br>
坏处：加密很慢，不能并行运算，必须要等前一块加密完后才能进行下一块的加密,另外，**cbc模式即使不知道IV,在只知道key的情况下，也能解除除初始块以外的plaintext**<br>

#### 3.3.3 CTR(Counter)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240712192919.png)
能并行计算，还能去除plaintext和ciphertext之间的关联！<br>