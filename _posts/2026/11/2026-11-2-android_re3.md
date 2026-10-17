---
layout: post
tags: [Android]
title: "android 应用tips3: frida一把梭"
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
    "lib": ["ES2020"],
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


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>