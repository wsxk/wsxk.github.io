---
layout: post
tags: [Android]
title: "android 应用tips"
author: wsxk
date: 2026-03-27
comments: true
---

- [1. 双开](#1-双开)
  - [1.1 双开原理](#11-双开原理)
  - [1.2 双开工具](#12-双开工具)
- [2. 汉化](#2-汉化)
  - [2.1 汉化步骤](#21-汉化步骤)
  - [2.2 汉化工具](#22-汉化工具)
- [3. 修改smali汇编](#3-修改smali汇编)
  - [3.1 np管理器修改smali代码](#31-np管理器修改smali代码)
- [4. 去广告](#4-去广告)
  - [4.1 广告类型](#41-广告类型)
  - [4.2 安卓组件](#42-安卓组件)
  - [4.3 activity生命周期](#43-activity生命周期)
  - [4.4 去广告弹窗的方法](#44-去广告弹窗的方法)
  - [4.5 去广告实操](#45-去广告实操)
- [references](#references)



简单记录一下android的使用小技巧。<br>
# 1. 双开<br>
## 1.1 双开原理<br>
安卓系统中通常以一个系统中`包名+用户（使用者身份）空间`来标识应用，所有绕过系统的双开限制，基本上围绕这两点来搞事：<br>

| 原理               | 解释                                                                                                                                     |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| 修改包名           |让手机系统认为这是2个APP，这样的话就能生成2个数据存储路径，此时的多开就等于你打开了两个互不干扰的APP                                                                                                                                          |
| 修改Framework      | 对于有系统修改权限的厂商，可以修改Framework来实现双开的目的，例如：小米自带多开                                                            |
| 通过虚拟化技术实现 | 虚拟Framework层、虚拟文件系统、模拟Android对组件的管理、虚拟应用进程管理 等一整套虚拟技术，将APK复制一份到虚拟空间中运行，例如：平行空间 |
| 以插件机制运行  |利用反射替换，动态代理，hook了系统的大部分与system—server进程通讯的函数，以此作为“欺上瞒下”的目的，欺骗系统“以为”只有一个apk在运行，瞒过插件让其“认为”自己已经安装。例如：VirtualApp|

## 1.2 双开工具<br>
`mt管理器和np管理器`,mt管理器的大部分功能都需要付费，所以这里用np管理器试验<br>
[http://normalplayer.top/](http://normalplayer.top/)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260322235404.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260322235458.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260322235550.png)
安装即可。<br>

# 2. 汉化<br>
顾名思义，将他国语言的app转为本国语言，通常**汉化包括asrc汉化、xml汉化和dex汉化**<br>
## 2.1 汉化步骤<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260323215230.png)

## 2.2 汉化工具<br>

在`np`管理器中，对提取的apk进行搜索功能，使用高级搜索，在文件中包含内容里寻找相关字符串：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260323210920.png)
找到字符串位置后，修改即可。<br>
针对看不懂的语言，可以用[开发者助手](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/%E5%BC%80%E5%8F%91%E8%80%85%E5%8A%A9%E6%89%8B.apk)来识图并修改。<br>
用开发者助手提取字符串，然后到`np`管理器里搜索对应的文本：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260323214154.png)


# 3. 修改smali汇编<br>
android应用通常运行在dalvik虚拟机/art虚拟机当中，文件格式为dex（可以类比linux的elf），汇编代码为smali（类比x86）。<br>
修改smali汇编的关键点在于快速定位代码位置，有**搜索关键字**和**抓取按钮id**两种方法。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260325231732.png)

## 3.1 np管理器修改smali代码<br>
定位到位置后，np管理器可以帮助我们方便的修改smali代码<br>
修改smali代码时有几个关键点需要注意：<br>
**.register xx标志了v（局部寄存器）和p（参数寄存器）的总数，且p总是在v的后面**<br>
**在smali里的所有操作都必须经过寄存器来进行:本地寄存器用v开头数字结尾的符号来表示，如v0、 v1、v2。 参数寄存器则使用p开头数字结尾的符号来表示，如p0、p1、p2。特别注意的是，p0不一定是函数中的第一个参数，在非static函数中，p0代指“this"，p1表示函数的第一个 参数，p2代表函数中的第二个参数。而在static函数中p0才对应第一个参数（因为Java的static方法中没有this方法）**<br>
修改完后保存，np管理器就会自动帮忙重新编译并打包签名。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260325234151.png)

# 4. 去广告<br>
## 4.1 广告类型<br>
广告类型有**启动广告（点开qq音乐有时候会有广告跳转到京东、淘宝等等）、弹窗/更新广告（弹个窗出来）、横幅广告（插在应用中**<br>
要想去除广告需要运用到anrdoid组件相关的知识。因为广告通常会有一个**activity组件**与其进行对应<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260329234445.png)

## 4.2 安卓组件<br>

| 组件                           | 描述                                                                                                                                                                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Activity(活动)                 | 在应用中的一个Activity可以用来表示一个界面，意思可以理解为“活动”，即一个活动开始，代表 Activity组件启动，活动结束，代表一个Activity的生命周期结束。一个Android应用必须通过Activity来运行和启动，Activity的生命周期交给系统统一管理。 |
| Service(服务)                  | Service它可以在后台执行长时间运行操作而没有用户界面的应用组件，不依赖任何用户界面，例如后台播放音乐，后台下载文件等。                                                                                                                 |
| Broadcast Receiver(广播接收器) | 一个用于接收广播信息，并做出对应处理的组件。比如我们常见的系统广播：通知时区改变、电量低、用户改变了语言选项等。                                                                                                                      |
| Content Provider(内容提供者)    |作为应用程序之间唯一的共享数据的途径，Content Provider主要的功能就是存储并检索数据以及向其他应用程序提供访问数据的接口。Android内置的许多数据都是使用Content Provider形式，供开发者调用的（如视频，音频，图片，通讯录等）                                                                                                                                                                                                                                       |

值得一提的是，安卓的应用的可视化界面（activity）都必须在`AndroidManifest`文件中进行申明，如下所示：<br>
```xml
        <!---声明实现应用部分可视化界面的 Activity，必须使用 AndroidManifest 中的 <activity> 元素表示所有 Activity。系统不会识别和运行任何未进行声明的Activity。----->
        <activity  
            android:label="@string/app_name"  
            android:name="com.zj.wuaipojie.ui.MainActivity"  
            android:exported="true">  <!--当前Activity是否可以被另一个Application的组件启动：true允许被启动；false不允许被启动-->
            <!---指明这个activity可以以什么样的意图(intent)启动--->
            <intent-filter>  
                <!--表示activity作为一个什么动作启动，android.intent.action.MAIN表示作为主activity启动--->
                <action  
                    android:name="android.intent.action.MAIN" />  
                <!--这是action元素的额外类别信息，android.intent.category.LAUNCHER表示这个activity为当前应用程序优先级最高的Activity-->
                <category  
                    android:name="android.intent.category.LAUNCHER" />  
            </intent-filter>  
        </activity>  
        <activity  
            android:name="com.zj.wuaipojie.ui.ChallengeFirst" />
        <activity  
            android:name="com.zj.wuaipojie.ui.ChallengeFifth"  
            android:exported="true" />  
        <activity  
            android:name="com.zj.wuaipojie.ui.ChallengeFourth"  
            android:exported="true" />  
        <activity  
            android:name="com.zj.wuaipojie.ui.ChallengeThird"  
            android:exported="false" />  
        <activity  
            android:name="com.zj.wuaipojie.ui.ChallengeSecond"  
            android:exported="false" />  
        <activity  
            android:name="com.zj.wuaipojie.ui.AdActivity" />  
```
**广告也是activity中的一种，所以代码中一定会有加载广告activity相关的逻辑**<br>

## 4.3 activity生命周期<br>
安卓对activity组件定义了各种生命周期:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260330221948.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260330222016.png)
## 4.4 去广告弹窗的方法<br>
```
1.xml中的versioncode ：有些弹窗根据版本是否为旧版本来弹，可以改versioncode来满足版本要求。

2.Hook弹窗(推荐算法助手开启弹窗定位) 

3.修改dex弹窗代码：定位到弹窗的启动代码，注释掉

4.抓包修改响应体(也可以路由器拦截) : 如果广告信息是从服务器来的，可以抓包
```

## 4.5 去广告实操<br>



# references<br>
[https://github.com/ZJ595/AndroidReverse/](https://github.com/ZJ595/AndroidReverse/)<br>
大佬写的真好~<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>