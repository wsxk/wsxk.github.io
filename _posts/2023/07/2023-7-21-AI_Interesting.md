---
layout: post
tags: [AI]
title: "AI interesting things"
author: wsxk
date: 2023-7-21
comments: true
---

- [前言](#前言)
- [1. 低资源复刻ChatGpt](#1-低资源复刻chatgpt)
- [2. AI村民组合成虚拟村庄](#2-ai村民组合成虚拟村庄)
- [3. 用LLM解释LLM](#3-用llm解释llm)
- [4. 让AI做计划自己运行自己](#4-让ai做计划自己运行自己)
- [5. 如何低价使用ChatGpt](#5-如何低价使用chatgpt)
  - [1. 缩短输入](#1-缩短输入)
  - [2. 自建模型](#2-自建模型)
  - [3. LLM cascade](#3-llm-cascade)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## 前言<br>
`ai`从出现后，特别是`ChatGpt`出现后，引导了许多热潮，直接导致很多有意思的AI用法的出现，有学者甚至用这些用法发了论文(anyway，这个事情是否合理就不过多讨论了，总之是发了)<br>
总之`递归`的思想在这些神奇的使用方法中得到了良好的诠释<br>
接下来来介绍一些神秘的`AI`用法<br>

## 1. 低资源复刻ChatGpt<br>
这个需求其实来自于一些人对于`ChatGpt`的不信任和数据安全的考量<br>
方法很简单，让`ChatGpt`全权包办就完事了<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723103256.png)
问题可以让`ChatGpt`想，任务什么的都让它想就完事了！<br>

## 2. AI村民组合成虚拟村庄<br>
这个是个有趣的实验，不过效果好像一般<br>
[https://arxiv.org/abs/2304.03442](https://arxiv.org/abs/2304.03442)论文在这<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723103705.png)

## 3. 用LLM解释LLM<br>
现在的做法，主要就是让chatgpt冒充某个模型中的一个神经元<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723103841.png)
让chatgpt看某个模型的神经元的运行输出，和输入样例，解释神经元的作用<br>
然后让chatgpt冒充那个神经元的输出，看看输出结果是否和原先一样<br>

## 4. 让AI做计划自己运行自己<br>
本质上是让AI分解大任务为小任务，然后一个一个做<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723104121.png)

## 5. 如何低价使用ChatGpt<br>
真穷人如何使用chatgpt？<br>
### 1. 缩短输入<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723104242.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723104258.png)

### 2. 自建模型<br>
可以参考低资源复刻chatgpt的案例<br>
或者考虑，把问过的问题缓存起来<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723104349.png)

### 3. LLM cascade<br>
不同模型的价格不同<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723104429.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723104418.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230723104450.png)