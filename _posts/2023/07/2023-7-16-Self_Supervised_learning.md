---
layout: post
tags: [AI]
title: "Self-supervised Learning"
date: 2023-7-16
author: wsxk
comments: true
---

- [Supervised vs Self-supervised](#supervised-vs-self-supervised)
- [Bert Self-supervised](#bert-self-supervised)
  - [1. Masking input](#1-masking-input)
  - [2. Next Sentence prediction](#2-next-sentence-prediction)
  - [3. Bert Application](#3-bert-application)
  - [4. extra](#4-extra)
- [Chatgpt](#chatgpt)
  - [1. Predict Next Token](#1-predict-next-token)
  - [2. How to use Chatgpt](#2-how-to-use-chatgpt)
- [Self-supervised extra](#self-supervised-extra)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## Supervised vs Self-supervised<br>
所谓的`supervised learning`,指的是传统的训练方法。即**有输入，也有标准输出作为对比**<br>
而`supervised learning`,它是没有标注输出的，**它把输入拆分成2个部分，一个部分用于模型的输入，另一部分用于模型的输出对比，它是没有标准答案的**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717135230.png)

## Bert Self-supervised<br>
`Bert`是采用了`Self-supervised`的方法训练得到的模型，之前说过，`Bert`做的是文字填空<br>
`Bert`使用的`self-supervised的方法`主要有2种<br>
### 1. Masking input<br>
将输入的一部分用某种符号表示，模型得到的结果和真正内容进行对比<br>
**顺道一提，把那个输入藏起来，用什么符号进行替换（mask或随机字符）都是随机的**<br>
**另外，对比也是采用的常用的交叉熵(cross-entropy)**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717154857.png)
### 2. Next Sentence prediction<br>
顾名思义，让`Bert`判断下一句是不是接在当前句子的后面的<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717155241.png)
不过很多实验表明，这个方法貌似没什么用，然而`SOP`作为变种取得了不错的结果<br>

### 3. Bert Application<br>
将`self-supervised Learning`预训练得到的bert模型，再进行微调(Finetune),得到可以满足各个子任务的模型<br>
**为什么Bert学会做填空题了之后就进行微调就可以得到很多专才呢，这之间有什么关联？我也不知道，后续或许就知道了**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717155342.png)
进行微调时，`Linear的参数是随机初始化的`，然而`Bert的初始化是设置好的`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717155522.png)
**事实表明，经过预训练的模型微调后的结果会比没有预训练的模型在同一数据集下，得到更好的表现**<br>
其实这也是很直觉的表现，毕竟预训练也事先也进行了很多训练<br>

### 4. extra<br>
事实证明，训练一个`Bert`是十分困难的，十分耗费资源<br>
为了减少消耗，还出现了关于`Bert 胚胎学`的学问，专门研究，`Bert`是在训练的那一个阶段获得了神奇的填空能力和自我学习能力<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717160022.png)
`Bert 是一个transformer encoder`，我们可以训练出一个相应的`Decoder`,采用的方法如下图所示，放入进行一些扰动后的输入，`Decoder`生成的输出应该跟完好的输入越接近越好<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717160251.png)

## Chatgpt<br>
`Chatgpt`的做法和`Bert`略有不同,使用的是方法是`Predict Next Token`<br>
### 1. Predict Next Token<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717160540.png)

### 2. How to use Chatgpt<br>
使用`Chatgpt`也和`Bert`不同，可能是因为`Chatgpt`模型真的太大了，以至于微调都很难进行，这就出现了之前说过的`In-Context Learning 和 Instruction tuning 和 Chain of Thought Prompting`<br>


## Self-supervised extra<br>
`Self-supervised 还有很多其他的做法和用途`,可见下图<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230717160832.png)