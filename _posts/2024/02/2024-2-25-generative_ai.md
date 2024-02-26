---
layout: post
tags: [AI]
title: "generative-ai 学习笔记"
author: wsxk
comments: true
date: 2024-2-25
---

- [前言](#前言)
- [0. 环境搭建](#0-环境搭建)
  - [0.1 申请API key](#01-申请api-key)
  - [0.2 安装python环境](#02-安装python环境)
  - [0.3 设置api key进行开发](#03-设置api-key进行开发)
- [1. generative ai 和 LLM的简单介绍](#1-generative-ai-和-llm的简单介绍)
  - [1.1 历史](#11-历史)
  - [1.2 LLM如何运作](#12-llm如何运作)
- [2. 探索并比较不同的LLM](#2-探索并比较不同的llm)


## 前言<br>
学习一下`generative ai`的课程，课程来自<br>
[https://github.com/microsoft/generative-ai-for-beginners](https://github.com/microsoft/generative-ai-for-beginners)<br>
PS：学习该课程属于是最近没事干了，又听闻AI的风声，蛮学着玩玩<br>

## 0. 环境搭建<br>
第0节环境搭建大家可自行参考文档，写得很详细，**甚至可以直接使用github提供的codespace功能一步到位**，这里附上我申请`open AI api`的方法<br>
### 0.1 申请API key<br>
[https://platform.openai.com/api-keys](https://platform.openai.com/api-keys)<br>
中创建 API key<br>
**当然，最重要的充值是不能忘记的**<br>

### 0.2 安装python环境<br>
```python
pip install openai
pip install python-dotenv
```

### 0.3 设置api key进行开发<br>
首先在`.gitignore`中写入一行`.env`，随后在你的工作区域下创建`.env`文件，其中写入你申请到的`api key`<br>
```
# Once you add your API key below, make sure to not share it with anyone! The API key should remain private.
OPENAI_API_KEY=sk-xxxx
```
然后再写代码<br>
```python
from openai import OpenAI
from dotenv import load_dotenv

# 将.env中的变量加载
load_dotenv()
client = OpenAI()

completion = client.chat.completions.create(
  model="gpt-3.5-turbo",
  messages=[
    {"role": "system", "content": "You are a poetic assistant, skilled in explaining complex programming concepts with creative flair."},
    {"role": "user", "content": "Compose a poem that explains the concept of recursion in programming."}
  ]
)

print(completion.choices[0].message)
```

## 1. generative ai 和 LLM的简单介绍<br>
### 1.1 历史<br>
ai的历史是比较久远的，**人工智能的第一个原型由打字的聊天机器人组成，依赖于从一组专家中提取并表示到计算机中的知识库。知识库中的答案是由输入文本中出现的关键字触发的。然而，很快人们就发现，这种使用打字聊天机器人的方法并不能很好地扩展。**<br>
20世纪90年代出现了机器学习：**能够从数据中学习模式，而无需显式编程。这种方法允许机器模拟人类语言理解：在文本标签配对上训练统计模型，使模型能够使用代表消息意图的预定义标签对未知输入文本进行分类。**<br>
近年来硬件发展迅速，又有了新的ai算法：神经网络<br>
**神经网络（特别是循环神经网络 - RNN）显着增强了自然语言处理，能够以更有意义的方式表示文本含义，并重视句子中单词的上下文。**<br>
又经过了数十年的研究:**一种名为Transformer 的新模型架构克服了 RNN 的限制，能够获得更长的文本序列作为输入。Transformer 基于注意力机制，使模型能够为其接收到的输入赋予不同的权重，“更加关注”最相关的信息集中的地方，而不管它们在文本序列中的顺序如何**,最近的`LLM`也是基于这种架构<br>

### 1.2 LLM如何运作<br>
输入是人类语言文本<br>
首先经过`tokenizer`：将人类语言文本分成若干个`token`，每个`token`是可变数量的字符组成的文本块，再把不同的`token`通过某种计算方式转换成数字<br>
其次经过`Predicting output tokens`步骤，**给定n个token作为输入，预测最有可能出现的下一个token**,预测出来后，把（n+1）个token作为输入，预测下一个可能出现的token，不断迭代<br>
最后经过`Selection process, probability distribution`步骤，模型会计算下一个token出现各个不同单词的概率，选择概率最大的那一个作为结果（然而并不总是选概率最大的，通过添加一定程度的随机性，让每次生成的输出都不一样）<br>
**LLM 是不确定性的，响应会有所不同，但是您可以通过参数设置来控制其方差。你也不应该期望它能完美地完成事情，它是来为你做繁重的工作的，这通常意味着你在需要逐渐改进的事情上得到了良好的第一次尝试。**<br>

## 2. 探索并比较不同的LLM<br>
本次目标为：<be>
```
1、了解当前不同类型的LLM ——> 知道如何选用合适的模型
2、测试、迭代、比较不同类型的模型（Azure） ——> 理解如何测试、迭代、提升LLM性能
3、如何部署LLM ——> 知道如何在商业中部署模型
```

