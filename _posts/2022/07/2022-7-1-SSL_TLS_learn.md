---
layout: post
title: "SSL/TLS协议"
date: 2022-7-1
tags: [web]
author: wsxk
comments: true
---

- [什么是SSL/TLS](#什么是ssltls)
  - [SSL/TLS的功能](#ssltls的功能)
  - [SSL/TLS的底层原理](#ssltls的底层原理)
- [TLS1.2握手过程](#tls12握手过程)
  - [ECDHE握手](#ecdhe握手)
    - [1.client hello](#1client-hello)
    - [2.server hello](#2server-hello)
    - [3.certificate](#3certificate)
    - [4.server key exchange](#4server-key-exchange)
    - [5.server hello done](#5server-hello-done)
    - [6.client key exchange](#6client-key-exchange)
    - [7.change cipher spec](#7change-cipher-spec)
    - [8.Finished(Encrypted Handshake Message)](#8finishedencrypted-handshake-message)
    - [9.change cipher spec](#9change-cipher-spec)
    - [10.Finished(Encrypted Handshake Message)](#10finishedencrypted-handshake-message)
  - [RSA握手](#rsa握手)
    - [1. client hello](#1-client-hello)
    - [2. server hello](#2-server-hello)
    - [3. certificate](#3-certificate)
    - [4. Server Hello Done](#4-server-hello-done)
    - [5. Client Key Exchange](#5-client-key-exchange)
    - [6. ChangeCipher Spec](#6-changecipher-spec)
    - [7. Finished](#7-finished)
    - [8. changeCipher Spec](#8-changecipher-spec)
    - [9. Finished](#9-finished)
  - [RSA握手和ECDHE握手的区别](#rsa握手和ecdhe握手的区别)
  - [RSA握手情况下知道服务器证书私钥后可以破解流量包](#rsa握手情况下知道服务器证书私钥后可以破解流量包)
  - [ECDHE握手情况下知道服务器证书私钥不可以破解流量包](#ecdhe握手情况下知道服务器证书私钥不可以破解流量包)
- [reference](#reference)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


# 什么是SSL/TLS<br>
SSL(secure socket layer)是进行安全传输的一种新型协议，建立在应用层和传输层之间。<br>
TLS(transport layer security)是SSL的升级版本<br>
总的来说，SSL/TLS是一个协议不同阶段的不同称呼（其实是一个东西）<br>
这个协议经历了SSL 1.0,SSL 2.0,SSL 3.0,TLS 1.0，TLS 1.1, TLS 1.2(目前最常用), TLS 1.3(目前最新)<br>

## SSL/TLS的功能<br>
SSL/TLS的功能主要如下:
> 1. 数据机密性: 保证传输的数据不会被窃听
> 2. 数据完整性: 保证传输的数据在传输过程中不会被篡改
> 3. 身份认证：保证进行传输的“目标”确实是你应该要传输的目标（即证明你妈是你妈）

总而言之，这玩意就是保护你传输数据的安全的。

## SSL/TLS的底层原理<br>
SSL/TLS 完全是建立在密码学的发展上的。<br>
即 非对称加密算法（由此诞生的证书和签名），对称加密算法 ，摘要算法<br>
可以去看看 《图解密码技术》。看完后，即使不知道算法的细节，也能理解SSL/TLS的运作机制<br>

# TLS1.2握手过程<br>
## ECDHE握手<br>
先看图
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220702210923.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220702210939.png)
### 1.client hello<br>
TLS连接由client开始<br>
client hello报文主要向服务器发送了 ***client生成的随机数（random_c）*** 、 ***可支持的加密套件（cipher suite）*** <br>
### 2.server hello<br>
server从client支持的加密套件中，选一套，并生成 ***session id（会话id）*** ，***服务端随机数（random_s)***<br>
然后发送给客户端<br>
使用ECDHE握手的话，会使用  ***TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256*** <br>
这是什么意思呢，RSA是进行身份认证时使用的算法，AES指的是对整个会话进行加密时使用的算法，SHA256指对会话进行hash（保证数据没被篡改），ECDHE指的是加密 AES的会话密钥 的方法<br>

### 3.certificate<br>
server把自己的证书链发送给client（client根据证书验证server是否可信）<br>
### 4.server key exchange<br>
这个过程中，服务器主要做了三件事<br>
> 1. 选择一条名为 name_curve的椭圆曲线，选好椭圆曲线相当于G点选好了，这些会公开给client
> 2. 生成随机数作为服务端椭圆曲线的私钥，保存在本地
> 3. 根据基点G和服务端椭圆曲线私钥生成服务端椭圆曲线公钥，发送给客户端

在发送包前，会使用摘要算法对这些公开信息进行哈希，并使用服务器证书的私钥进行签名（防止篡改）<br>  
### 5.server hello done<br>
server发送server hello done，告知client 相关信息发送完毕<br>
### 6.client key exchange<br>
在client验证完成server端证书的合法性后，会生成一个随机数作为client端的椭圆曲线私钥。并根据server发送的信息生成client端的椭圆曲线公钥。<br>
并将自己的公钥发送给server端。<br>
至此，双方都有对方的椭圆曲线公钥、自己的椭圆曲线私钥和公钥、椭圆曲线基点 G。于是，双方都就计算出点（x，y），其中 x 坐标值双方都是一样的，前面说 ECDHE 算法时候，说 x 是会话密钥，但实际应用中，x 还不是最终的会话密钥。<br>
TLS 握手阶段，客户端和服务端都会生成了一个随机数传递给对方<br>
最终的会话密钥，就是用「客户端随机数 + 服务端随机数 + x（ECDHE 算法算出的共享密钥） 」三个材料生成的。<br>
之所以需要3个，是因为客户端或服务器「伪随机数」的可靠性不可信，为了保证真正的完全随机，把三个不可靠的随机数混合起来，那么「随机」的程度就非常高了<br>
### 7.change cipher spec<br>
告知server以后会使用加密会话（换句话说其实前6个都是明文，但是不会有安全问题），其实这个报文没什么用<br>
### 8.Finished(Encrypted Handshake Message)<br>
其实也没什么用，就是用来check 生成的公钥（这里指ECDHE公钥）是否正确
### 9.change cipher spec<br>
与7一致<br>
### 10.Finished(Encrypted Handshake Message)<br>
同8

## RSA握手<br>
### 1. client hello<br>
有client端使用的TLS 版本号、支持的密码套件列表，以及生成的随机数（Client Random）<br>
### 2. server hello<br>
server端选择 密码套件列表，生成随机数(server random)<br>
在RSA握手过程中，选择的密码套件很有可能是 ***Cipher Suite:  TLS_RSA_WITH_AES_128_GCM_SHA256*** <br>
### 3. certificate<br>
服务端为了证明自己的身份，会发送「Server Certificate」给客户端，这个消息里含有数字证书<br>
### 4. Server Hello Done<br>
表示server信息传输完毕.<br>
### 5. Client Key Exchange<br>
首先客户端验证证书，证书验证通过后，继续后面的操作。<br>
客户端产生一个新的随机数 (pre-master)，并使用服务端的RSA公钥加密该随机数，通过「Change Cipher Key Exchange」消息传给服务端。该随机数就是后面会话密钥的重要组成成分之一<br>
### 6. ChangeCipher Spec<br>
告诉server开始加密会话<br>
### 7. Finished<br>
客户端再发一个「EncryptedHandshake Message（Finished）」消息，把之前所有发送的数据做个摘要，再用会话密钥（master secret）加密一下，让服务器做个验证，验证加密通信是否可用，以及验证之前握手信息是否有被中途篡改过。会话密钥（master secret）为randomClient+randomServer+ pre-master。<br>
### 8. changeCipher Spec<br>
同6<br>
### 9. Finished<br>
同7

## RSA握手和ECDHE握手的区别<br>
RSA 密钥协商算法「不支持」前向保密，ECDHE 密钥协商算法「支持」前向保密；<br>
使用了 RSA 密钥协商算法，TLS 完成四次握手后，才能进行应用数据传输，而对于 ECDHE 算法，客户端可以不用等服务端的最后一次 TLS 握手，就可以提前发出加密的 HTTP 数据，节省了一个消息的往返时间；<br>
使用 ECDHE， 在 TLS 第 2 次握手中，会出现服务器端发出的「Server Key Exchange」消息，而 RSA 握手过程没有该消息；<br>

## RSA握手情况下知道服务器证书私钥后可以破解流量包<br>
由前面的流程可以看到，RSA密钥交换过程中，是客服端选择一个随机数作为会话密钥，然后用服务端证书的公钥加密，加密后的密文传输过去，然后服务端用私钥解密。

表面看是实现了密钥交换，但实际上，会话密钥还是在网络中进行了传输，因此每次数据表中都可以得到。得到后如果有服务端的证书和私钥，就可以解密了。

因此也称此为不具有前向安全性，只要服务端不换证书，那么所有证书范围内的会话都可以进行解密。对于旁路监听流量，拥有全量数据包的情况下，是可以全部解密的

## ECDHE握手情况下知道服务器证书私钥不可以破解流量包<br>
由前面的流程可以看到，与RSA密钥协商算法不同的是，ECDHE在进行会话密钥协商时，第2和第3次握手中，都是服务端与客服端生成自己的临时公私钥对，在网络中交换时，仅仅只是传输了公钥，会话密钥完全在本地计算，而且双方的私钥也未暴露在网络中，所以只是抓包和知道证书与私钥，也是不能恢复出会话密钥的

# reference<br>
[https://wonderful.blog.csdn.net/article/details/77866773](https://wonderful.blog.csdn.net/article/details/77866773)<br>
[https://blog.csdn.net/s493197604/article/details/104612970](https://blog.csdn.net/s493197604/article/details/104612970)<br>
[https://www.nuomiphp.com/serverfault/zh/604782192a6fd71e7017e2b2.html](https://www.nuomiphp.com/serverfault/zh/604782192a6fd71e7017e2b2.html)<br>
[https://www.cnblogs.com/20179203li/p/7875451.html](https://www.cnblogs.com/20179203li/p/7875451.html)<br>
[图解 ECDHE 密钥交换算法](https://blog.csdn.net/m0_50180963/article/details/113061162)<br>
[一文读懂https中密钥交换协议的原理及流程](https://weibo.com/ttarticle/p/show?id=2309634765862589498097)