---
layout: post
tags: [re]
title: "frida hook"
author: wsxk
date: 2022-10-6
comments: true
---

- [前置条件<br>](#前置条件)
- [安装frida<br>](#安装frida)
- [firda使用<br>](#firda使用)

## 前置条件<br>
有root权限的android手机: [https://wsxk.github.io/pixel2xl%E5%88%B7%E6%9C%BA/](https://wsxk.github.io/pixel2xl%E5%88%B7%E6%9C%BA/)<br>

## 安装frida<br>
```python
pip install frida
pip install frida-tools
```
安装完后，进入frida官网[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)下载符合你frida版本的`firda server`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221006193210.png)
下载后，将server使用`adb push` 命令传送到手机上并添加root权限<br>

## firda使用<br>
可以使用`am start -n com.droidlearn.activity_travel/com.droidlearn.activity_travel.FlagActivity`选择要运行的控件类<br>
可以通过Manifest文件获取`package名`和`name`
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221006193739.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221006193806.png)

运行脚本如下:<br>
```python
import frida

device = frida.get_usb_device()
session = device.attach(23992) #进程pid
with open(r"C:\Users\wsxk\Documents\python\flag2.js",encoding='UTF-8') as f:
    script = session.create_script(f.read())
script.load()
# 脚本会持续运行等待输入
input()
```
js文件编写:<br>
```javascript
console.log("Script loaded successfully ");
Java.perform(function x() {
    console.log("Inside java perform function");
    //定位类
    var my_class = Java.use("com.droidlearn.activity_travel.FlagActivity"); 
    console.log("Java.Use.Successfully!");
    //在这里更改类的方法的实现（implementation）
    my_class.access$004.implementation = function (x){
        //打印替换前的参数
        my_class.access$004.call(this,x);
        x.cnt.value = x.cnt.value+11000;
        console.log("Successfully!");
        console.log(x.cnt.value);
        
        console.log("after flag")
        return x.cnt.value;
    }
    var access$000 = my_class.access$000;
    access$000.implementation = function(x){
        console.log(x.cnt.value);
        return access$000.call(this,x);
    }

    var Toast = Java.use("android.widget.Toast");
    var makeText = Toast.makeText;
    var String = Java.use("java.lang.String");
    // 函数重载, 设置参数类型
    makeText.overload("android.content.Context", "java.lang.CharSequence", "int").implementation = function (
      context,
      content,
      time
    ) {
      console.log(content);
      // 设置字符串重复次数
      var content = "wsxk yyds!\n".repeat(10);
      // 实例化字符串
      var hookContent = String.$new(content);
      // 可以hook, 但是不能打断原先的程序, 原来该做什么, 还要继续做下去
      //return this.makeText(context, hookContent, time);
      return makeText.call(this,context,hookContent,time);
    };
});
```