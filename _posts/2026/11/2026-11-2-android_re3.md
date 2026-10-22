---
layout: post
tags: [Android]
title: "android 应用tips3: frida17+一把梭"
author: wsxk
date: 2026-11-2
comments: true
---

- [前言](#前言)
- [1. frida 17以上版本的差异点](#1-frida-17以上版本的差异点)
  - [1.1 代码补全](#11-代码补全)
- [2. frida 语法](#2-frida-语法)
  - [2.1 附加调用：Java.perform](#21-附加调用javaperform)
  - [2.2 java层hook: java.use和java.choose](#22-java层hook-javause和javachoose)
  - [2.3 native层hook： interceptor.attach](#23-native层hook-interceptorattach)
    - [2.3.1 native层 inline hook](#231-native层-inline-hook)
  - [2.4 native层主动调用: NativeFunction](#24-native层主动调用-nativefunction)
- [3. JNI\_Onload的执行逻辑](#3-jni_onload的执行逻辑)
- [4. PC手机调试指南](#4-pc手机调试指南)
  - [4.1 桌面显示: scrcpy](#41-桌面显示-scrcpy)
  - [4.2 adb 安装app应用](#42-adb-安装app应用)
  - [4.3 python注入frida脚本开发（frida rpc）](#43-python注入frida脚本开发frida-rpc)
- [references](#references)


# 前言<br>
frida先前的尝试可参考:<br>
[pixel 2xl 刷机+root一把梭](https://wsxk.github.io/pixel2xl%E5%88%B7%E6%9C%BA/)<br>
[frida hook](https://wsxk.github.io/frida_hook/)<br>
[frida hook revenge](https://wsxk.github.io/frida/)<br>

# 1. frida 17以上版本的差异点<br>
frida更新还是很快到，到了17及以上版本，版本变更导致安装过程发生变化，主要是从`javascript`转换到`typescript`<br>
详见[[原创]关于frida17.0.0以上版本的pythonAPI运行js脚本导致bridge问题](https://bbs.kanxue.com/thread-287941-1.htm)<br>

## 1.1 代码补全<br>
工作目录下创建执行命令<br>
```
npm install frida-java-bridge # 新增
npm install @types/frida-gum
```
当前工作目录下创建`tsconfig.json`：<br>
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2020","DOM"],
    "types": ["frida-gum"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true
  },
  "include": ["./*.ts"],
  "exclude": ["node_modules", "dist"]
}
```
在写hook脚本前，需要引入如下库:<br>
```typescript
import Java from "frida-java-bridge"
```


# 2. frida 语法<br>
## 2.1 附加调用：Java.perform<br>
```
Java.perform: 本质是在我们想要操纵的应用中添加一个线程，这个线程用来执行我们写的frida脚本函数。所以每次写frida脚本都必须以这个函数开始来写。
```

## 2.2 java层hook: java.use和java.choose<br>
```
Java.use: Java.use用于获取类的定义
Java.choose: Java.choose用于获取类在Java VM 中的实例化对象
```
要想灵活使用Java.use和Java.choose，关键在于理解java中的类的 静态变量/实例变量，静态函数/实例函数<br>
```
1. 静态变量：
静态变量在类的定义中，由static标识
静态变量在java类对象的所有实例中共享，即A类中有静态变量B，A的实例C设置B的值为1后，A的实例D使用B的值也为1. 

2. 实例变量：
类定义中，没有static表示的变量均为实例变量。
实例变量每个java类对象的实例均有自己的独立副本。

3. 静态函数：
静态函数在类的定义中，也由static标识
静态方法可以通过 类名 直接调用，而不需要创建该类的实例。也可以通过类的实例来调用，但推荐通过类名调用。
静态方法只能访问类的静态变量和静态方法。它不能直接访问实例变量和实例方法，因为静态方法是在类加载时加载的，而实例变量和实例方法依赖于特定的实例。

4. 实例函数：
实例方法没有 static 关键字
实例方法可以访问静态变量、静态方法、实例变量和实例方法
实例方法需要通过 类的实例 来调用，必须先创建类的对象，然后通过该对象调用实例方法。

```
所以他们两个函数的用法:<br>
```
1. Java.use: 获取android应用中类的定义。
这只是获得了类的定义，并没有获得类的实例对象；
因此在这种情况下能做只有直接调用静态方法，修改静态变量，以及对类定义的方法进行implementation hook。

2. Java.choose: 获取android应用中已有的类的实例对象.
得到一个类的实例对象，可以直接调用静态方法，修改静态变量，调用动态方法，修改动态变量。
```

## 2.3 native层hook： interceptor.attach<br>
```
Interceptor.attach: hook native层函数，即so中的函数。
```

### 2.3.1 native层 inline hook<br>
相当于篡改某段程序执行的逻辑。<br>
```js
function inline_hook() {
    var soAddr = Module.findBaseAddress("lib52pojie.so");
    if (soAddr) {
        var func_addr = soAddr.add(0x10428);
        Java.perform(function () {
            Interceptor.attach(func_addr, {
                onEnter: function (args) {
                    console.log(this.context.x22); //注意此时就没有args概念了
                    this.context.x22 = ptr(1); //赋值方法参考上一节课
                },
                onLeave: function (retval) {
                }
            }
            )
        })
    }
}
```
某段代码解析为汇编:<br>
```js
var soAddr = Module.findBaseAddress("lib52pojie.so");
var codeAddr = Instruction.parse(soAddr.add(0x10428));
console.log(codeAddr.toString());
```

直接patch某段代码:<br>
```js
var soAddr = Module.findBaseAddress("lib52pojie.so");
var codeAddr = soAddr.add(0x10428);
Memory.patchCode(codeAddr, 4, function(code) {
const writer = new Arm64Writer(code, { pc: codeAddr });
writer.putBytes(hexToBytes("20008052"));
writer.flush();
});
function hexToBytes(str) {
var pos = 0;
var len = str.length;
if (len % 2 != 0) {
    return null;
}
len /= 2;
var hexA = new Array();
for (var i = 0; i < len; i++) {
    var s = str.substr(pos, 2);
    var v = parseInt(s, 16);
    hexA.push(v);
    pos += 2;
}
return hexA;
}
```
[https://armconverter.com/](https://armconverter.com/) arm转hex在线工具。<br>

## 2.4 native层主动调用: NativeFunction<br>
```js
var funcAddr = Module.findBaseAddress("lib52pojie.so").add(0x1054C);
//声明函数指针
//NativeFunction的第一个参数是地址，第二个参数是返回值类型，第三个[]里的是传入的参数类型(有几个就填几个)
var aesAddr = new NativeFunction(funcAddr , 'pointer', ['pointer', 'pointer']);
var encry_text = Memory.allocUtf8String("OOmGYpk6s0qPSXEPp4X31g==");    //开辟一个指针存放字符串       
var key = Memory.allocUtf8String('wuaipojie0123456'); 
console.log(aesAddr(encry_text ,key).readCString());
```


# 3. JNI_Onload的执行逻辑<br>
[https://wsxk.github.io/android_re2/](https://wsxk.github.io/android_re2)
里提到了android应用加载so的逻辑，只到`JNI_Onload`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20261020000356.png)


# 4. PC手机调试指南<br>

## 4.1 桌面显示: scrcpy<br>
[https://github.com/Genymobile/scrcpy](https://github.com/Genymobile/scrcpy)<br>
```
scrcpy --video-codec=h265 --max-size=1920 --max-fps=60 --no-audio --keyboard=uhid
```
```
参数	作用
scrcpy	启动 Android 设备投屏及控制工具
--video-codec=h265	使用 H.265 视频编码，压缩效率高、画质好，但可能增加延迟
--max-size=1920	限制投屏画面最长边不超过 1920 像素，保持原始宽高比
--max-fps=60	限制最高帧率为 60 FPS，使画面更流畅（不保证达到 60 FPS）
--no-audio	禁用音频转发，只传输画面
--keyboard=uhid	模拟物理 USB HID 键盘，支持更完整的键盘输入和快捷键
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20261021233826.png)


## 4.2 adb 安装app应用<br>
```
adb install -r xxx.apk
```


## 4.3 python注入frida脚本开发（frida rpc）<br>
（本质开发python脚本去自动化得注入js/ts脚本）<br>
```python

import frida

from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent

def on_message(message, data):
    if message["type"] == "send":
        print("[Agent]", message["payload"])
    elif message["type"] == "error":
        print("[Error]", message.get("stack", message))
    else:
        print("[Message]", message)


# 1. 编译 TypeScript
compiler = frida.Compiler()
compiler.on(
    "diagnostics",
    lambda diag: print("[Compile]", diag)
)
bundle = compiler.build(
    str(BASE_DIR / "hook.ts"),
    project_root=str(BASE_DIR)
)
print("[+] TypeScript compiled")

# 2. 获取 USB 设备
device = frida.get_usb_device(timeout=5)
# 3. 附加到目标进程
session = device.attach(27146) # 通过frida-ps -U 找到对应进程id，attach上去
print("[+] Attached")

# 4. 创建并加载 Agent
script = session.create_script(bundle)
script.on("message", on_message)
script.load()

print("[+] ts injected")


# 5. 保持注入会话
try:
    input("Press Enter to exit...\n")
finally:
    session.detach()
```

ts脚本模板如下:<br>
```typescript
import Java from "frida-java-bridge"

Java.perform(()=>{
    console.log("hello");    
    send("hhhh");
})
```




# references<br>
[《安卓逆向这档事》十五、是时候学习一下Frida一把梭了(下)](https://www.52pojie.cn/thread-1840174-1-1.html)<br>
[《安卓逆向这档事》十六、是时候学习一下Frida一把梭了(终)](https://www.52pojie.cn/thread-1859820-1-1.html)<br>



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>