---
layout: post
tags: [Android]
title: "Android Reverse Engineering"
author: wsxk
comments: true
date: 2023-10-20
---

- [1. android逆向简介](#1-android逆向简介)
- [2. 通过mainifest.xml文件查看入口](#2-通过mainifestxml文件查看入口)
- [3. java层逆向](#3-java层逆向)
- [4. native层逆向](#4-native层逆向)
  - [4.1 静态注册](#41-静态注册)
  - [4.2 动态注册](#42-动态注册)
  - [4.3 .init段分析](#43-init段分析)
- [5. 代码混淆](#5-代码混淆)
  - [5.1 办法1： ida findcrypt3](#51-办法1-ida-findcrypt3)
  - [5.2 办法2： 反混淆工具](#52-办法2-反混淆工具)
- [6. 反调试机制](#6-反调试机制)
- [references](#references)


## 1. android逆向简介<br>
总的来说，`android`逆向可以分为`java层逆向`和`native层逆向`<br>
`java层逆向`就是很简单的，**把一个apk文件拖入Android分析工具（比如jeb、jadx）中，就可以看到java代码**<br>
`native层逆向`指的是**把代码存入一个so文件（通常由c/c++编写），然后再把so文件打包进apk中，在apk中会调用native层的代码，然而一般的Android逆向工具无法看到其代码，需要将so文件拖入ida进行分析**<br>

## 2. 通过mainifest.xml文件查看入口<br>
举个例子：
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" xmlns:app="http://schemas.android.com/apk/res-auto" android:versionCode="1" android:versionName="1.0" package="com.example.mobicrackndk">
    <uses-sdk android:minSdkVersion="8" android:targetSdkVersion="17" />
    <application android:theme="@style/AppTheme" android:label="@string/app_name" android:icon="@drawable/ic_launcher" android:allowBackup="true">
        <activity android:label="@string/app_name" android:name="com.example.mobicrackndk.CrackMe">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```
其中的`activity android:label`的这个项，即是入口点<br>

## 3. java层逆向<br>
一般用`jeb`，`jadx`反编译出来的就是java层代码，需要你自己熟读代码<br>

## 4. native层逆向<br>
`java层想要调用native层的函数，其实so文件中是有迹可循的,即被调用的函数必须被JNI注册`<br>
**JNI注册方法分为静态注册和动态注册，静态注册的方法可以在IDA的函数窗口或者导出表中直接找到，比较简单。动态注册的方法需要分析JNI_OnLoad函数**<br>
### 4.1 静态注册<br>
静态注册的方法可以在IDA的函数窗口或者导出表中直接找到，比较简单<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20231029001749.png)

### 4.2 动态注册<br>
动态注册的方法需要分析JNI_OnLoad函数<br>
以下是一个简单的`JNI_OnLoad`函数<br>
```c
//第一步，实现JNI_OnLoad方法
JNIEXPORT jint JNI_OnLoad(JavaVM* jvm, void* reserved){
    //第二步，获取JNIEnv
    JNIEnv* env = NULL;
    if(jvm->GetEnv((void**)&env, JNI_VERSION_1_6) != JNI_OK){
        return JNI_FALSE;
    }
    //第三步，获取注册方法所在Java类的引用
    jclass clazz = env->FindClass("com/curz0n/MainActivity");
    if (!clazz){
        return JNI_FALSE;
    }
    //第四步，动态注册native方法
    if(env->RegisterNatives(clazz, gMethods, sizeof(gMethods)/sizeof(gMethods[0]))){
        return JNI_FALSE;
    }
    return JNI_VERSION_1_6;
}
```
其中第四步gMethods变量是JNINativeMethod结构体，用于映射Java方法与C/C++函数的关系，其定义如下:
```c
typedef struct {
    const char* name; //动态注册的Java方法名
    const char* signature; //描述方法参数和返回值
    void*       fnPtr; //指向实现Java方法的C/C++函数指针
} JNINativeMethod;
```
所以逆向中，动态注册的函数经常可以在`Jni_onload`里发现端倪<br>
我们知道JNI_OnLoad函数的第一个参数是JavaVM指针类型，这里IDA工具不能自动识别，所以需要手动修复一下，选中int，右键选择Set lvar tyep(快捷键Y)重新设置变量类型,这有助于帮你理解代码<br>

### 4.3 .init段分析<br>
在链接so共享目标文件的时候，如果so中存在.init和.init_array段，则会先执行.init和.init_array段的函数，然后再执行JNI_OnLoad函数。通过静态分析可知，JNI_OnLoad函数中的v4指针指向的地址上的变量值是加密状态，在实际运行的过程中，v4指针指向的地址上的值应该是解密状态，所以解密的操作应该在JNI_OnLoad函数运行之前，.init或者.init_array段上的函数。
查看Segments视图（快捷键Ctrl+S），该目标文件只存在.init_array段:
**有时候init段会进行一些加解密操作，需要注意**<br>

## 5. 代码混淆<br>
android逆向常规步骤如上述，先看manifest.xml文件，找到入口点，然后看java层代码，再看native层代码。<br>
然而，事情不总是那么顺利，有的人比较恶心啊，会给代码进行混淆，这样你看不懂到底是什么逻辑。<br>
### 5.1 办法1： ida findcrypt3<br>
[https://github.com/polymorf/findcrypt-yara](https://github.com/polymorf/findcrypt-yara)<br>
android的反编译工具没有findcrypt3,怎么办呢？其实`findcrypt`的原理是，**搜索一些加密算法中会使用到的常数来判断使用了什么加密算法**<br>
所以，如果遇到了混淆的代码，我们可以 ***抄出里面用到的常数，编译成x86架构的执行文件，再放入ida中使用findcrypt进行分析！*** <br>

### 5.2 办法2： 反混淆工具<br>
这里列两个目前用过比较顶的java反混淆工具：<br>
[https://seosniffer.com/javascript-deobfuscator](https://seosniffer.com/javascript-deobfuscator)<br>
[https://github.com/kuizuo/js-deobfuscator](https://github.com/kuizuo/js-deobfuscator)<br>


## 6. 反调试机制<br>
有的apk会有反调试机制，比如检测是否被`frida`、`xposed`、`cydia`等hook框架hook，如果被hook了，就会直接退出<br>
`frida`详情可参考[https://wsxk.github.io/frida_hook/](https://wsxk.github.io/frida_hook/)<br>
这里列举一个文章，这篇文章教你如何绕过神秘的反调试：<br>
[https://bbs.kanxue.com/thread-277034.htm](https://bbs.kanxue.com/thread-277034.htm)<br>

## references<br>
[https://curz0n.github.io/2021/05/10/android-so-reverse/#0x00-%E5%89%8D%E8%A8%80](https://curz0n.github.io/2021/05/10/android-so-reverse/#0x00-%E5%89%8D%E8%A8%80)<br>
[https://ctf-wiki.org/android/basic_reverse/static/so-example/](https://ctf-wiki.org/android/basic_reverse/static/so-example/)<br>