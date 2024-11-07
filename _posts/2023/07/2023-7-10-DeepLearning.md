---
layout: post
tags: [AI]
title: "Deep Learning&Gradient Descent& Backpropagation"
date: 2023-7-10
author: wsxk
comments: true
---

- [Deep Learning](#deep-learning)
  - [1. Fully Connect Feedforware Network](#1-fully-connect-feedforware-network)
  - [2. 本质](#2-本质)
- [Gradient Descent](#gradient-descent)
  - [1. Learning Rate](#1-learning-rate)
  - [2. Stochastic Gradient Descent](#2-stochastic-gradient-descent)
  - [3. Feature Scaling](#3-feature-scaling)
  - [4. Gradient Descent的数学定理](#4-gradient-descent的数学定理)
  - [5. Gradient Descent的限制](#5-gradient-descent的限制)
- [Backpropagation](#backpropagation)
  - [1. 基本原理](#1-基本原理)
  - [2. Forward pass](#2-forward-pass)
  - [3. Backward pass](#3-backward-pass)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## Deep Learning<br>
`Deep Learning`其实也服从常规的计算方法<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712123244.png)
而且之前的介绍也说过，`Deep Learning`其实是`Logistic Regression`的堆叠<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712123801.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712123837.png)
### 1. Fully Connect Feedforware Network<br>
`Fully Connect Feedforware Network`是深度学习中一种常见的链接方式。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712124031.png)

### 2. 本质<br>
**本质上，深度学习是矩阵运算**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712124226.png)
使用`GPU`可以加速。<br>
并且，其实很符合`Logistic Regression`的逻辑<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712124309.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712124533.png)
其实还有一个最重要的问题:<br>
**深度学习，Hidden Layer越多越好，其实是一个很直觉的结果；即使不用深度学习，参数（parameters）越多，模型效果比较好是很正常的；那么用深度学习有什么优势吗?**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230712124802.png)

## Gradient Descent<br>
`Gradient Descent`有许多小tips要学习。<br>
### 1. Learning Rate<br>
首先关于`Learning Rate`,必须要调整其为合适的值。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713075511.png)
自己手动调整比较麻烦，于是出现了自动调整的办法<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713075834.png)
比较好用的是叫做`Adagrad`的办法,原理如下图所示。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713080208.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713080523.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713080618.png)
**Adagrad是当前比较稳定的方法**<br>
`Adagrad为什么比较好呢`?<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713082003.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713082410.png)

### 2. Stochastic Gradient Descent<br>
这是一个很神奇的`Gradient Descent`方法,主打的就是一个唯快不破<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713082906.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713094442.png)

### 3. Feature Scaling<br>
这个技术的使用场景发生在：不同权重对`Loss Function`的影响不一样，导致做`Gradient Descent`效率不够快时。如果每个权重的影响相同，那么做`Gradient Descent`时，效率十分快。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713100830.png)
一种常见的办法是<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713101512.png)

### 4. Gradient Descent的数学定理<br>
基于泰勒公式<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713102615.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713103015.png)
举个实际的例子<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713104139.png)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713104235.png)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713104322.png)<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713103947.png)<br>

### 5. Gradient Descent的限制<br>
**会卡在`Local minima`(局部最低点)的地方**<br>
实际上更严重，因为实际计算时，基本上算不出来微分是0的位置。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230713104743.png)

## Backpropagation<br>
**Backpropagation，其实是更有效率的计算Gradient Descent**<br>
在一个`neural network`里，大模型参数都有几百万，算起来太复杂了。于是就有了这个东西。<br>
### 1. 基本原理<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-07-14%20125729.png)
接下来给一个实际例子。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-07-14%20130215.png)
通过上述的规则转换，得到如下结果<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-07-14%20130533.png)
### 2. Forward pass<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-07-14%20130639.png)

### 3. Backward pass<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-07-14%20131612.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230714141223.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230714141252.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-07-14%20131701.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230714141054.png)
