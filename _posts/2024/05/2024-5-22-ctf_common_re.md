---
layout: post
tags: [re]
title: "Re 常见加解密算法识别与加解密脚本"
date: 2024-5-22
author: wsxk
comments: true
---

- [0. findcrypt3](#0-findcrypt3)
- [1.  古典加密算法](#1--古典加密算法)
  - [1.1 caesar: 凯撒密码](#11-caesar-凯撒密码)
  - [1.2 vigenere](#12-vigenere)
- [2. base系列](#2-base系列)
  - [2.1 base64](#21-base64)
  - [2.2 base32](#22-base32)
- [3. TEA](#3-tea)
- [4. RC4](#4-rc4)
- [5. AES](#5-aes)
  - [5.1 AES加解密脚本](#51-aes加解密脚本)
- [6. MD5](#6-md5)


<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

## 0. findcrypt3<br>
https://github.com/polymorf/findcrypt-yara<br>
好用的ida插件，帮助你快速识别算法。<br>
当然了，有时候这个插件也不一定能帮助你找到这个问题，所以你需要知道一些常见算法的特征<br>

## 1.  古典加密算法<br>
### 1.1 caesar: 凯撒密码<br>
凯撒密码的加密、解密可以通过取模的加减法进行计算。首先将字母用数字替代 A = 0,B = 1, …, Z = 25。 当偏移量为n的时候加密方法是<br>

$$ 
c = m + n {\ }mod {\,}26 
$$

解密方法是：<br>

$$
m = c - n {\ } mod {\,}26
$$

其加解密脚本为:<br>
```python

key = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'
shift = 3

def encrypt_caesar(plaintext, shift):
    ciphertext = ''
    for char in plaintext:
        if char in key:
            ciphertext += key[(key.index(char) + shift) % 26]
        else:
            ciphertext += char
    return ciphertext

def decrypt_caesar(ciphertext, shift):
    return encrypt_caesar(ciphertext, -shift)


data = "THE QUICK BROWN FOX JUMPS OVER THE LAZY DOG"
encrypt = encrypt_caesar(data, shift)
print(encrypt)
print(decrypt_caesar(encrypt,shift))
```

### 1.2 vigenere<br>
viginere加密是从caesar引申而来的加密方式:<br>

$$
c_i = m_i + k_i {\;} mod {\;} 26
$$

其实是把固定的值`n` 替换成了一组密钥`k`<br>
解密方式:<br>

$$
m_i = c_i - k_i {\;} mod {\;} 26
$$

其加解密脚本:<br>
```python

alpha = 'abcdefghijklmnopqrstuvwxyz'
key = 'hhhhh'

def encrypt_vigenere(plain_text, key):
    encrypted_text = ''
    key_length = len(key)
    key_index = 0
    for char in plain_text:
        if char in alpha:
            shift = alpha.index(key[key_index])
            encrypted_text += alpha[(alpha.index(char) + shift) % 26]
            key_index = (key_index + 1) % key_length
        else:
            encrypted_text += char
    return encrypted_text

def decrypt_vigenere(encrypted_text, key):
    decrypted_text = ''
    key_length = len(key)
    key_index = 0
    for char in encrypted_text:
        if char in alpha:
            shift = alpha.index(key[key_index])
            decrypted_text += alpha[(alpha.index(char) - shift) % 26]
            key_index = (key_index + 1) % key_length
        else:
            decrypted_text += char
    return decrypted_text

data = "attackatdawn"
encrypted_data = encrypt_vigenere(data, key)    
print(encrypted_data)
decrypted_data = decrypt_vigenere(encrypted_data, key)
print(decrypted_data)
```

## 2. base系列<br>
### 2.1 base64<br>
如果在程序中出现了`base64`的索引表，大概率是用了base64，有些人可能会对base64的表进行部分更换，问题也不大。<br>
```python
import base64

origin_charset = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
#custom_charset = b"ZYXWVUTSRQPONMLKJIHGFEDCBAzyxwvutsrqponmlkjihgfedcba9876543210+/"
custom_charset = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
def base64_encode(data):
    return base64.b64encode(data).translate(bytes.maketrans(origin_charset, custom_charset))

def base64_decode(data):
    return base64.b64decode(data.translate(bytes.maketrans(origin_charset, custom_charset)))

data = b"attackatdawn"
encoded_data = base64_encode(data)
print(encoded_data)
decoded_data = base64_decode(encoded_data)
print(decoded_data)
```
### 2.2 base32<br>
也是一种加密方式，只不过相比于`base64`而言，字符更少:<br>
```python
import base64
#输入的数据必须是比特流
s = b'aaaaa'
enc = base64.b32encode(s)
print(enc)
print(base64.b32decode(enc))
```

## 3. TEA<br>
关于TEA算法的识别，如果程序中出现了固定常数`0x9e377969/0x61c88647`，那么很有可能是tea加密或其变种

## 4. RC4<br>
常见的流加密方式。识别方式为，初始代码中会对大小为256的表进行赋值和交换操作<br>

## 5. AES<br>
对`aes`的识别首先需要知道AES加密的步骤。<br>
首先是根据key生成轮密钥。
> 1初始化：即 明文 和 密钥 作异或
> 前9轮：
> > 一.字节替换（SubBytes）: 用s盒对 输入进行字节替换<br>
> > 二.行移位（ShiftRows）：输入化成4 * 4矩阵，第i行循环左移i个字节（i=0，1，2，3）<br>
> > 三.列混淆（MixColumns）：输入化成4 * 4矩阵，乘以固定的矩阵<br>
> > 四.轮密钥加（AddRoundKey）：输入与轮密钥矩阵异或<br>
> 
> 第10轮: 字节替换、行移位、轮密钥加

因此如果代码中发现了s盒（s盒可以参加仓库的aes的c代码）,可以判断是aes加密，具体模式需要自己判断<br>

### 5.1 AES加解密脚本<br>
以`AES ECB`模式为例<br>
```python
from Crypto.Cipher import AES

key = b'1234567890123456'
aes = AES.new(key, AES.MODE_ECB)

# Encrypt
data = b'Hello, world!!!!'
encrypt=aes.encrypt(data)
print(encrypt)

# decrypt
decrypt=aes.decrypt(encrypt)
print(decrypt)
```

## 6. MD5<br>
md5加密的加密步骤通常如下：<br>
```c
md5_ctx sample
md5_init(&sample)
md5_update_string(&sample,plain)
md5_final(digest,&sample)
```
在md5_init函数中，会出现初始化赋值`0x67452301 0xefcdab89 0x98badcfe 0x10325476`，即有可能是md5加密


