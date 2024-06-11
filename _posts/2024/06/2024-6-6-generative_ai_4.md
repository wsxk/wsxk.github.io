---
layout: post
tags: [AI]
title: "generative-ai 学习笔记 Ⅳ"
author: wsxk
comments: true
date: 2024-6-6
---

- [前言](#前言)
- [10. Building Low Code AI Applications](#10-building-low-code-ai-applications)
- [11. Integrating with function calling](#11-integrating-with-function-calling)
  - [11.1 Why Function Calling](#111-why-function-calling)


## 前言<br>
请看完前面的章节，anyway其实你不看也没什么大问题~<br>
😄<br>

## 10. Building Low Code AI Applications<br>
目前认识的接触过所谓低代码的朋友都对它嗤之以鼻，让我来鉴定一下他的成分！<br>
目前来看，所谓低代码，指的是**让你不需要开发代码，或者使用很少的代码来构建应用程序**，类似于少儿编程的那种东西，让你通过拖拽某些代表功能的积木，来搭建你想要的程序。<br>
anyway，我感觉这好像没什么好学的，pass。<br>

## 11. Integrating with function calling<br>
### 11.1 Why Function Calling<br>
用函数调用的方式来集成`AI`功能（anyway，已经说了很多遍了，感觉这也有点水了）。<br>
优势如下：<br>
```
Consistent response format : 一致的响应格式
如果我们可以更好的控制响应格式，我们就能够更容易的把响应集成到下游的其它系统中。


External data : 拓展的数据
可以在一个对话中，使用来自于其他来源的数据，可以解决LLM训练时数据的时间限制
```

