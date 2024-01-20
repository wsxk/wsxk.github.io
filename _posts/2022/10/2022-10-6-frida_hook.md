---
layout: post
tags: [Android]
title: "frida hook"
author: wsxk
date: 2022-10-6
comments: true
---

PS:`修改于2024-1-20`<br>

- [1. frida：真机调试](#1-frida真机调试)
  - [1.1 前置条件](#11-前置条件)
  - [1.2 安装frida](#12-安装frida)
  - [1.3 firda使用](#13-firda使用)
- [2. frida：模拟器调试](#2-frida模拟器调试)

## 1. frida：真机调试<br>
### 1.1 前置条件<br>
有root权限的android手机: [https://wsxk.github.io/pixel2xl%E5%88%B7%E6%9C%BA/](https://wsxk.github.io/pixel2xl%E5%88%B7%E6%9C%BA/)<br>

### 1.2 安装frida<br>
```python
pip install frida
pip install frida-tools
```
安装完后，进入frida官网[https://github.com/frida/frida/releases](https://github.com/frida/frida/releases)下载符合你frida版本的`firda server`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221006193210.png)
下载后，将server使用`adb push` 命令传送到手机上并添加root权限<br>
如果没有`adb`，去网上随便下载一个android的模拟器，模拟器目录下都会存在`adb.exe`执行程序,把其所在目录加入到环境变量当中即可使用<br>

### 1.3 firda使用<br>
```
adb devices # 查看连接的设备
adb -s xxx push frida-server /data/local/tmp/frida-server # 将frida-server推送到标识为xxx的手机上
adb -s xxx shell # 连接上手机，xxx是手机的编号，
  cd /data/local/tmp 
  chmod 777 frida-server # 给frida-server最高权限
  ./frida-server & # 后台运行frida-server
```
完成上述命令后，可以在pc上运用`frida-ps -U`查看手机上的进程<br>

可以使用`am start -n com.droidlearn.activity_travel/com.droidlearn.activity_travel.FlagActivity`选择要运行的控件类<br>
可以通过Manifest文件获取`package名`和`name`
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221006193739.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221006193806.png)

运行脚本如下:<br>
```python
import frida

device = frida.get_usb_device(timeout=10) #设置一下时间，如果不设时间，可能导致超时 然后找不到设备
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

## 2. frida：模拟器调试<br>
这里采用**夜神模拟器**对的模拟终端进行测试<br>
使用步骤和`1.2节`开始后的步骤一样<br>
