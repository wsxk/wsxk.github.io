---
layout: post
tags: [AI]
title: "Finetune & Prompt"
author: wsxk
date: 2023-7-15
comments: true
---

- [前言:对于大语言模型的期待](#前言对于大语言模型的期待)
  - [1. Chatgpt vs Bert](#1-chatgpt-vs-bert)
- [专才(Bert)](#专才bert)
  - [1. 加外挂](#1-加外挂)
  - [2. 微调参数（Finetune）](#2-微调参数finetune)
  - [3. adapter](#3-adapter)
- [通才(chatgpt)](#通才chatgpt)
  - [1. In-Context Learning](#1-in-context-learning)
  - [2. Instruction tuning](#2-instruction-tuning)
  - [3. Chain of Thought(COT) prompting](#3-chain-of-thoughtcot-prompting)


## 前言:对于大语言模型的期待<br>
讲到大语言模型，除了当今比较流行的`Chatgpt`以外，还有一个重量级项目，就是`Bert`<br>
### 1. Chatgpt vs Bert<br>
其实`chatgpt`本质上是文字接龙，而`Bert`本质上是文字填空，这就衍生成了我们对大模型的2个不同期待：<br>
**专才(Bert)和通才(chatgpt)**<br>
专才很容易理解，就是让大模型解决某一个特定的任务，比如说翻译、提取摘要。<br>
通才也很容易理解，就是让大预言模型能够做到各种事情，比如又能做翻译，又能提取摘要，你想让它做啥它就做啥。<br>
**专才的好处就是，专才在某一特定任务的执行情况下，通常会比通才做的更好**<br>
**通才的好处就是，重新设计prompt(让模型做什么事)，就可以快速地开发新功能，不用再写程序了**<br>

## 专才(Bert)<br>
对于专才，通常的做法是在预训练模型`Bert`的基础上做一些改造<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716180256.png)
从图中可以看到有2种改造方法，加外挂和微调参数<br>
### 1. 加外挂<br>
其实外挂顾名思义，是一些`Bert`中有提供的办法，这里不做详细了解<br>
### 2. 微调参数（Finetune）<br>
顾名思义，做微调<br>
从头开始训练模型时，`initialization`的设置是随机的，要通过训练，得到较好的结果。<br>
我们在已经训练好的模型的基础上再进行一些训练，就叫做微调,其实**finetune本质上还是在做gradient descent**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716180527.png)
### 3. adapter<br>
adapter本质上其实还是微调参数的办法<br>
大概就是，如果你对一整个模型做微调，参数还是比较多的。<br>
但是你给这个模型额外设置一些参数，当成一个`adapter`，你在微调的时候只需要训练`adapter`中的额外参数就好了，比较省事<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716180740.png)
`adapter`的另一个好处是，如果你要在一个大模型上开发100个不同的专项模型，你可能需要保存100个训练好的专项模型,如果模型很大，就很占空间啦<br>
**如果使用adapter的话，你只要保留一个大模型和100个adapter就好了，adapter比模型小很多，能够节省资源**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716181037.png)


## 通才(chatgpt)<br>
chatgpt朝着通才的方向努力，一个模型能够做很多事情（如果你有用chatgpt，你也能感受到它的强力）<br>
为了让模型能够做通才，有很多训练办法<br>
**主要思路有，让模型读范例(In-context Learning)和让模型读题目描述(Instruction tuning)**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716181531.png)
### 1. In-Context Learning<br>
让模型读范例然后做其他题目的方法，叫做`In-context Learning`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716182502.png)
其实这个方法还遭到了质疑，在小模型情况下，可能学不到什么东西，然而在大模型的情况下，这个方法还是有效果的<br>

### 2. Instruction tuning<br>
当模型试图理解题目表达的含义，叫做`Instruction tuning`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716182708.png)<br>

### 3. Chain of Thought(COT) prompting<br>
这个方法非常的神奇啊，大概就是让大模型回答问题的时候，让它输出推理过程（如何得到答案的），大模型的回答正确率就大大提高了。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716182857.png)
这个方法还有许多的配套方法<br>
比如`Self-consistency`让模型重复推导同一个问题的答案，选一个最多被推导的答案作为最终答案输出<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716183257.png)
还有叫做`Least-to-most-prompting`的方法，这个方法让模型学会将一个问题拆解成小问题来解决<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716183343.png)<br>
还有一个神秘的方法，让模型自己寻找最合适的`prompting`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230716183533.png)<br>
