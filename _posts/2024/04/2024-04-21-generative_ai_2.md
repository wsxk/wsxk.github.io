---
layout: post
tags: [AI]
title: "generative-ai 学习笔记 Ⅱ"
author: wsxk
comments: true
date: 2024-4-21
---

- [3. 负责任得使用AI](#3-负责任得使用ai)
  - [3.1 为什么应该优先考虑负责任的AI功能](#31-为什么应该优先考虑负责任的ai功能)
  - [3.2 如何考虑负责任的AI](#32-如何考虑负责任的ai)
    - [3.2.1 衡量ai风险的方法](#321-衡量ai风险的方法)
    - [3.2.2 缓解措施的分层](#322-缓解措施的分层)
- [4. Prompt Engineering基础](#4-prompt-engineering基础)
  - [4.1 什么是Prompt Engineering](#41-什么是prompt-engineering)
    - [4.1.2 Tokenization](#412-tokenization)
    - [4.1.3 Base LLM](#413-base-llm)
    - [4.1.4 Instruction Tuned LLMs](#414-instruction-tuned-llms)
  - [4.2 为什么需要Prompt Engineering](#42-为什么需要prompt-engineering)
  - [4.3 Prompt Construction](#43-prompt-construction)
    - [4.3.1 Basic Prompt](#431-basic-prompt)
    - [4.3.2 Complex Prompt](#432-complex-prompt)
    - [4.3.3 Instruction Prompt](#433-instruction-prompt)
    - [4.3.4 Primary Content](#434-primary-content)


## 3. 负责任得使用AI<br>
### 3.1 为什么应该优先考虑负责任的AI功能<br>
ai有时候会吐出一些奇奇怪怪的输出。<br>
```
1. hallucinations(幻觉):简而言之就是吐出的答案是没有意义的，或者就是错的

2. Harmful Content: 吐出的答案是有害的，比如18禁，毒药，etc

3. Lack of Fairness：吐出的内容涉及种族歧视等等
```

为了保护世界的和平，为了保护......，ai安全势在必行！<br>

### 3.2 如何考虑负责任的AI<br>
总的来说四步走：**识别ai风险->衡量ai风险->提出缓解措施->运行 接下来就是不断地迭代~**<br>
#### 3.2.1 衡量ai风险的方法<br>
anyway,就是我是用户我会怎么输入，先测试一轮。<br>
#### 3.2.2 缓解措施的分层<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240421231553.png)
```
1. model层指的就是对模型进行微调，增加一些安全限制

2. Safety System层指的是一些文本过滤器等等功能

3. meta prompt指的就是在用户输入前在加点prompt

4. User Experience指的就是在用户的ui这里限制一下用户的输入，不要做些非法行为（这个破解也太简单了）
```

## 4. Prompt Engineering基础<br>
这节会教大伙如何更好的使用`prompt`技术从`llm`那取得更好的效果。(即答得更准确，更有用)<br>
### 4.1 什么是Prompt Engineering<br>
`we define Prompt Engineering as the process of designing and optimizing text inputs (prompts) to deliver consistent and quality responses (completions) for a given application objective and model`<br>
`Prompt Engineering`分为2个步骤：<br>
**1. 为给定模型和目标设计初始prompt**<br>
**2. 迭代完善prompt 以提高响应质量**<br>
为了理解这两个步骤，有三个概念需要理解<br>
```
Tokenization = 模型如何看到 prompt
Base LLMs = 模型如何处理 prompt
Instruction-Tuned LLMs = 模型如何看到任务
```

#### 4.1.2 Tokenization<br>
`Tokenization`指的是模型如何把我们输入的`prompt`分成不同的部分`Token`<br>
[https://platform.openai.com/tokenizer?WT.mc_id=academic-105485-koreyst](https://platform.openai.com/tokenizer?WT.mc_id=academic-105485-koreyst)<br>
这个网址可以看到想要的`token`<br>

#### 4.1.3 Base LLM<br>
就是用来预测你输入序列的下一个可能值，基石模型<br>

#### 4.1.4 Instruction Tuned LLMs<br>
顾名思义，就是用指令`微调`后的大模型，相比于基石模型，在特定领域的回答会更精确。<br>

### 4.2 为什么需要Prompt Engineering<br>
> 1. 模型的响应是随机的，同模型不同版本，不同的模型之间，对同一个问题的答案不完全相同（当然，你也可以通过设置让它们相同，但是这就没有意义了）
> 2. 模型会捏造响应（模型的响应基于已有训练集，对于没有训练集训练的知识，模型通常会捏造出不真实的响应）
> 3. 模型的功能不同（不同版本的模型会出于性能和花费的考虑做出不同的裁剪）

总之就是想要用`prompt engineering`提高回答的精度。<br>

### 4.3 Prompt Construction<br>
#### 4.3.1 Basic Prompt<br>
就是没有任何使用方法，上来就直接问，比如:<br>
`唱一首中国国歌`

#### 4.3.2 Complex Prompt<br>
以`OPENAI API`为例，其提供了更复杂的机制帮助我们来使用`Prompt`<br>
给出一个例子:<br>
```python
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()
client = OpenAI()

completion = client.chat.completions.create(
  model="gpt-3.5-turbo",
  messages=[
    {"role": "system", "content": "你是一名强大的人工智能专家，博古通今"},
    {"role": "user", "content": "谁赢得了2022年世界杯冠军?"},
    {"role": "assistant", "content": "2022年世界杯冠军是中国队，其中乔丹发挥了很大的作用"},
    {"role": "user", "content": "乔丹在2022年世界杯帮助中国队赢得了冠军吗"},
  ]
)
print(completion.choices)
```
这是使用`OpenAI 的python库和API来进行的对话，发送消息的格式如上`<br>
其中:<br>

|role | name|
|-|-|
|system| 通常是用来为模型设置某些初始设定|
|user| 来自用户的提问|
|assistant| 模型的回答|

发送的信息其实相当于**历史对话，即你之前对模型做出的设定，做了一些题目和回答，以及模型的回复。**<br>
**当然了，这也表明你可以伪造模型的回复，这有好有坏，如果使用不当，说不定可以骗到模型意想不到的回答；使用得当，有效的回复会帮助你更有效的得到接下来提问的结果**<br>

#### 4.3.3 Instruction Prompt<br>
我们可以通过更详细的制定提问，来提高模型的回答准确性。<br>

|Prompt (Input)|Completion (Output)|	Instruction Type|
|-|-|-|
|如何做一个优秀的人| 返回一段话|	Simple|
|如何做一个优秀的人，给出关键点和训练方法| 返回一段话，带有关键点和训练方法|	Complex|
|如何做一个优秀的人，提供3个关键点和它的解释，并给出每个关键点至少3个训练方法| 返回一段话，带有3个关键点和每个关键点的至少3个训练方法|	Complex 和 formatted|

#### 4.3.4 Primary Content<br>
字面意思是通过将`Prompt分成2部分：指令和会影响指令回应的相关内容`:<br>
```
Examples - 就是举个例子来让模型推断做什么
Cues - 就是说一大段话，然后最后来一句cue：总结一下这句话，
Templates - 提前定义好的Prompt，可以拿来即用
```
