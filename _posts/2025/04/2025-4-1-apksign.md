---
layout: post
tags: [Android]
title: "修改apk文件重新打包"
author: wsxk
date: 2025-4-1
comments: true
---

- [0. 写在前面](#0-写在前面)
- [1. 环境准备](#1-环境准备)
  - [1.1 java](#11-java)
  - [1.2 apktool](#12-apktool)
  - [1.3 android build-tools](#13-android-build-tools)
- [2. 实操环节](#2-实操环节)
  - [2.1 apk解包](#21-apk解包)
  - [2.2 修改解包目录中的内容](#22-修改解包目录中的内容)
  - [2.3 重打包](#23-重打包)
  - [2.4 zipalign 4字节对齐](#24-zipalign-4字节对齐)
  - [2.5 apk签名](#25-apk签名)
    - [2.5.1 keytool 生成证书文件](#251-keytool-生成证书文件)
    - [2.5.2 apksign生成签名后的apk文件](#252-apksign生成签名后的apk文件)
  - [2.6 验证](#26-验证)
- [references](#references)


# 0. 写在前面<br>
做android逆向的时候遇到了一个麻烦的题目，如果能够修改so文件的话就可以快速解决，奈何修改so后不知道如何重新打包android应用，现在来找补一下<br>

# 1. 环境准备<br>
## 1.1 java<br>
我个人是在java8和java17之间纠结过一阵子的，直到了解到sprint-boot选用Java17作为它的主流支持版本，说明java8要被渐渐淘汰了，因此安装版本选用java17<br>
进入oracle官网下载java17版本即可。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20250324211813.png)
下载版本后直接运行完毕。<br>
**注意，java17安装完毕后，记得修改环境变量，msi安装会默认帮你在系统变量上添加一个路径，但是我们不要用它，将其修改成你实际的java安装路径，比如说我的安装路径是C:\Program Files\Java\jdk-17\bin**<br>
修改后如下图所示:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20250324212045.png)
**这么做的原因是保持默认无法使用java17中自带的某些签名工具，比如keytool**<br>

## 1.2 apktool<br>
安卓逆向的实用工具，解包、重打包的利器，进入[https://github.com/iBotPeaches/Apktool](https://github.com/iBotPeaches/Apktool)下载最新版本即可<br>

## 1.3 android build-tools<br>
这个看绝大多数的blog，都是从[https://www.androiddevtools.cn/](https://www.androiddevtools.cn/)中下载的，不知道为什么，我无法访问这个网址，从[https://github.com/inferjay/AndroidDevTools](https://github.com/inferjay/AndroidDevTools)中尝试下载工具的百度网盘链接都过期了，有点难绷<br>
退而求其次，去`android studio`官网里下载。[https://developer.android.com/studio?hl=zh-cn](https://developer.android.com/studio?hl=zh-cn)<br>
我们不需要安装android studio，只需要它的命令行工具就行了因此:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20250324212705.png)
但是下载完后并不等于结束，将下载的压缩包解压后，进入bin目录，执行如下命令:<br>
```shell
.\sdkmanager.bat --sdk_root=./ --install "build-tools;34.0.0"
# 执行sdkmanager命令，安装目录在当前目录下，安装build-tools，至于后面的34.0.0怎么来的
# 请看链接https://developer.android.com/tools/releases/build-tools?hl=zh-cn
```
**下载完成后记得将其添加到环境变量中**<br>

# 2. 实操环节<br>
## 2.1 apk解包<br>
```shell
java -jar .\apktool_2.11.1.jar d .\app1.apk -o app1
```
该命令能够将app1.apk解包并把内容写到app1目录当中。<br>
## 2.2 修改解包目录中的内容<br>
这一部分模拟修改apk文件的操作，我添加了一个`1.txt`文件<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20250324213327.png)

## 2.3 重打包<br>
```shell
java -jar .\apktool_2.11.1.jar b .\app1\ -o app1_new.apk
```
该命令能够把app1目录重新打包成apk，命名为app1_new.apk

## 2.4 zipalign 4字节对齐<bvr>
这个是在android studio官网里看到的内容[https://developer.android.com/tools/zipalign?hl=zh-cn](https://developer.android.com/tools/zipalign?hl=zh-cn)<br>
```shell
zipalign.exe -f 4 .\app1_new.apk app1_new_zipaling.apk
```

## 2.5 apk签名<br>
### 2.5.1 keytool 生成证书文件<br>
```shell
keytool -genkeypair -keyalg RSA -keysize 2048 -validity 10000 -keystore mykeystore.jks
# keytool是java自带的密钥和证书管理工具
# -genkeypair生成密钥对，即公钥和私钥 
# -keyalg RSA 使用RSA生成密钥 
# -keysize 2048 密钥长度2048
# -validity 10000 有效期10000天
# -keystore mykeystore.jks 私钥和公钥存放在mykeystore.jks文件中
```
里面的内容随便填填就好了。**密钥库口令你要记得**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20250324214021.png)

### 2.5.2 apksign生成签名后的apk文件<br>
```shell
apksigner sign -ks .\mykeystore.jks -out app1_signed.apk .\app1_new_zipaling.apk
```
输入你的passwd即可完成生成<br>


## 2.6 验证<br>
将app1_signed.apk拖入模拟器当中，可以正常运行:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-9-25/20250324214235.png)

# references<br>
[Android11.0 生成系统签名.jks文件并对Apk进行签名](https://blog.csdn.net/wxd_csdn_2016/article/details/131300689?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7EPayColumn-1-131300689-blog-105138228.235%5Ev43%5Econtrol&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7EPayColumn-1-131300689-blog-105138228.235%5Ev43%5Econtrol&utm_relevant_index=1)<br>
[Android之通过 apksigner 对 apk 进行 手动签名](https://blog.csdn.net/q610098308/article/details/105138228)<br>
[apksigner生成签名](https://tool.yimenapp.com/info@-apksigner-sheng-cheng-qian-ming-147708.html)<br>
[zipalign安卓优化工具安装](https://blog.csdn.net/qq_39965424/article/details/134717735)<br>
[https://developer.android.com/tools/apksigner?hl=zh-cn](https://developer.android.com/tools/apksigner?hl=zh-cn)<br>
[https://developer.android.com/tools/releases/build-tools?hl=zh-cn](https://developer.android.com/tools/releases/build-tools?hl=zh-cn)<br>
[https://github.com/iBotPeaches/Apktool](https://github.com/iBotPeaches/Apktool)<br>
[【Android 逆向】修改 Android 的 apk 安装包内的文件并重新打包 ( apktool_2.6.0.jar 下载和使用 | zipalign 文件对齐 | apksigner 签名 )](https://cloud.tencent.com/developer/article/2253134)<br>
[Android APK重打包](https://blog.csdn.net/kelxLZ/article/details/125823384)<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>