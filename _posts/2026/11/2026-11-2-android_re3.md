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
  - [2.1](#21)


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
## 2.1 



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>