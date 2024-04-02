---
layout: post
tags: [Android]
title: "android 安全加固原理"
date: 2024-4-02
author: wsxk
comments: true
---

- [前言](#前言)
- [5. Android加固原理](#5-android加固原理)
  - [5.1 Android原理图](#51-android原理图)
  - [5.2 dex文件格式](#52-dex文件格式)
  - [5.3 Application类](#53-application类)
  - [5.4 android加固流程](#54-android加固流程)
  - [5.5 常见加固平台以及安全加固的优劣](#55-常见加固平台以及安全加固的优劣)
- [references](#references)


## 前言<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/M_7V%7EPSYXZARLHONMU%2483%25C.jpg)
老规矩，看这篇前，建议了解一下我之前写得内容（<br>
[https://wsxk.github.io/android_vm/](https://wsxk.github.io/android_vm/)<br>

## 5. Android加固原理<br>
### 5.1 Android原理图<br>
目前大多安全加固的原理都如下图所示:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240402193119.png)
### 5.2 dex文件格式<br>
说到安全加固，不得不了解dex的文件格式，先前的文章讲过，dex文件存放的是运行在`Dalvik虚拟机内的字节码`<br>
其整体结构如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240402191112.png)

如果用`010 editor`查看某个`dex`文件的话，会发现如下结构：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240402191157.png)
基本对应<br>

| 数据名称 | 功能解释 |
|:------:|:------:|
|dex_header | dex文件头部记录整个dex文件的相关属性 |
|string_ids| 记录一些字符串常量的索引|
|type_ids|记录了android中的类的字符串名称索引|
|proto_ids| 记录函数的返回值，参数等等信息的索引|
|field_ids|记录类的field名称的索引|
|method_ids| 记录类的method名称的索引|
|class_def| 记录类的定义及名称的索引|
|data| 数据区，保存了各个类的真实数据|
|link_data|静态链接数据区|

`dex_header`中，有几个字符需要重点关注：**checksum、signature、fileSize**<br>
> 1. checksum: 使用alder32算法校验从该字段开始（不包括checksum本身）到文件末尾（即从文件的第12字节开始到文件末尾）的完整性
> 2. signature：使用SHA-1算法校验文件的完整性（checksum发现错误，就无需进行SHA-1校验，也算是双重保险）
> 3. fileSize ： 记录dex文件大小

在使用加固技术对dex文件进行加固后，这三个字段是必须要修改的！！！<br>

### 5.3 Application类<br>
**Application类比程序中的其他类启动的都要早，因此在分析Android程序中，需要先查看该程序是否具有Application类，如果有，就要看看它的oncreate()方法是否做了一些影响逆向分析的初始化工作**<br>
`Application`和`Activity`，`Service`一样是Android框架的一个系统组件，当Android程序启动时系统会创建一个Application对象，用来存储系统的一些信息。Android系统自动会为每个程序运行时创建一个Application类的对象且只创建一个，所以Application可以说是单例（singleton）模式的一个类。<br>
绝大部分加壳apk，都会在`application`类做些文章<br>
通常我们是不需要指定一个Application的，系统会自动帮我们创建，如果需要创建自己的Application，创建一个类继承Application并在AndroidManifest.xml文件中的application标签中进行注册（只需要给application标签增加name属性，并添加自己的 Application的名字即可）。
```java
<application
        android:name="CustomApplication">
</application>
```
启动Application时，系统会创建一个PID，即进程ID，所有的Activity都会在此进程上运行。那么我们在Application创建的时候初始化全局变量，同一个应用的所有Activity都可以取到这些全局变量的值，换句话说，我们在某一个Activity中改变了这些全局变量的值，那么在同一个应用的其他Activity中值就会改变。<br>

Application对象的生命周期是整个程序中最长的，它的生命周期就等于这个程序的生命周期。因为它是全局的单例的，所以在不同的Activity,Service中获得的对象都是同一个对象。所以可以通过Application来进行一些，如：数据传递、数据共享和数据缓存等操作。<br>

### 5.4 android加固流程<br>
对于一个加固后的apk，启动时经过的步骤如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240402194453.png)

对于一个刚刚生成的`未加固`的apk，其加固流程如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240402194520.png)
主要流程如下：<br>
> 1. 加密阶段: 对原dex文件进行加密得到encrypt.dex文件
> 2. 合成阶段：将encrypt.dex文件添加到壳dex文件的文件末尾，并修改壳dex文件头部中的 checksum、signature、filesize三个字段，得到新的classes.dex文件
> 3. 修改原apk文件并重新打包签名：修改classes.dex为新的classes.dex，修改Androidmanifest.xml文件的启动配置
> 4. 运行新的apk程序

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240402195302.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240402195334.png)

### 5.5 常见加固平台以及安全加固的优劣<br>
常见的加固平台有：`梆梆加固，爱加密，360加固，腾讯加固`<br>
至于安全加固的优劣，其实也比较明显:<br>

    　　正面:

    　　　　1.保护自己核心代码算法,提高破解/盗版/二次打包的难度

    　　　　2.缓解代码注入/动态调试/内存注入攻击

    　　负面：

    　　　　1.影响兼容性

    　　　　2.影响程序运行效率.

    　　　　3.部分流氓、病毒也会使用加壳技术来保护自己

    　　　　4.部分应用市场会拒绝加壳后的应用上架

## references<br>
[https://www.cnblogs.com/bmjoker/p/11831479.html](https://www.cnblogs.com/bmjoker/p/11831479.html)<br>