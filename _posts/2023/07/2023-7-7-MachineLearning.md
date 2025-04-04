---
layout: post
tags : [AI]
title: "Machine learning basic"
date: 2023-7-7
author: wsxk
comments: true
---

- [什么是机器学习](#什么是机器学习)
  - [1. 回归(Regression)](#1-回归regression)
  - [2. 分类(Classification)](#2-分类classification)
  - [3. 生成式学习(Generative Learning)](#3-生成式学习generative-learning)
- [寻找函数的三步骤](#寻找函数的三步骤)
  - [1. 找出函数集合(Model)](#1-找出函数集合model)
  - [2. 订出评价函数好坏的标准(Loss Function)](#2-订出评价函数好坏的标准loss-function)
  - [3. 找出最好的函数](#3-找出最好的函数)
  - [总结](#总结)
  - [Extra](#extra)
- [Generative Learning](#generative-learning)
  - [1. AR（Autoregressive Model）](#1-arautoregressive-model)
  - [2. NAR（Non-autoregressive Model）](#2-narnon-autoregressive-model)
  - [3. AR vs NAR](#3-ar-vs-nar)
  - [4. AR/NAR 结合](#4-arnar-结合)
- [Total Training Process](#total-training-process)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## 什么是机器学习<br>
**机器学习，本质上是让机器自动找一个函数**<br>
比如`ChatGPT`，就是要找一个函数，输入是`一个句子`，输出是`句子后可能跟着的句子`。<br>
比如`Midjourney`,它找的函数就是，输入是`一段描述`，输出是`一张图片`<br>
比如`AlphaGo`,它找的函数是，输入是`围棋棋盘的信息`，输出是`下一次棋子的落子`<br>
以函数输出分类的话，机器学习可以分为3种<br>
### 1. 回归(Regression)<br>
即输出是一个数值<br>
### 2. 分类(Classification)<br>
即输出是一个类别，就是做选择题<br>
### 3. 生成式学习(Generative Learning)<br>
也叫`Structed Learning`,就是生成有结构的物件，比如（影像，文句）<br>
`ChatGpt`在原理上属于`Classification`,在用户体验上属于`Generative Learning`,相当于用分类的办法实现生成式学习<br>

## 寻找函数的三步骤<br>

### 1. 找出函数集合(Model)<br>
深度学习中的类神经网络结构（例如`CNN`,`RNN`,`Transformer`等等），指的都是候选函数集合<br>

### 2. 订出评价函数好坏的标准(Loss Function)<br>
就是之前提到的`Loss Function`<br>
这里可以根据你训练资料的不同做不同的`Function`,比如，如果你用的是`Supervised Learning`,即，每个数据都有想要的专家标注，可以制定简单的`Loss Function`<br>
如果你使用的是`Semi-supervised Learning`,即只有部分数据有标注，就需要较复杂的`Loss Function`,提高预测度。<br>

### 3. 找出最好的函数<br>
使用比如之前说到的`Gradient Descent`,找到一个最好的函数<br>

### 总结<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202023-07-10%20103023.png)
**这里需要了解到，不同的什么学习，在机器学习中的定位是什么**<br>

### Extra<br>
在第3步中，虽然理论上可以找到最好函数，实际上，并不总是能跑出来<br>
**所以在实际中，我们需要自己设定Learning Rate，Batch Size、初始化方法,等等。为了区分机器学习内部代码的参数，这些人为操控的参数被称为超参数(Hyperparameter)**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230710120616.png)
另外在第2步中,有时候在训练数据集上面效果好的函数，在测试数据集上面不一定好，**很大程度上是因为训练集不够多，容易过拟合,因此需要在Loss上做额外考量**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230710121050.png)<br>

在第1步中，为了减少函数集合的数量，提高筛选效率,通常会设定一个合理范围内的函数集合。**然而减少了函数集合范围，容易错过较好的函数，如何合理定制函数集合，也是一门技术活**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230710121343.png)

最后，**其实某些步骤的额外考虑在这个步骤里面不一定会有作用，但是可以帮助到其他步骤。**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230710121445.png)

## Generative Learning<br>
这里详细介绍一下`Generative Learning`<br>
先前讲过，这里指的是生成有结构的复杂物件，比如文句、影像、语言<br>
`Generative Learning`中有2个主要方法**AR（Autoregressive Model）和NAR（Non-autoregressive Model）**<br>

### 1. AR（Autoregressive Model）<br>
俗称`逐个击破`<br>
用生成句子来理解，就是一次生成一个字，直到遇到停止符号。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230711104902.png)
### 2. NAR（Non-autoregressive Model）<br>
俗称`一次到位`<br>
同样还是生成句子来理解，就是一次想把生成的字都生成了。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230711104950.png)
但是此时会遇到如何停止的问题。<br>
大体有2个办法**比如预设500长度，让Model生成500长度，把停止符号后的文字全仍了；或者无论如何，都生成100长度的文字**<br>

### 3. AR vs NAR<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230711105152.png)

### 4. AR/NAR 结合<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230711105233.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230711105250.png)


## Total Training Process<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/MachineLearning.jpg)