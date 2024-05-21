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
    - [4.3.5 Supporting Content](#435-supporting-content)
    - [4.3.4 总结](#434-总结)
- [5. Advanced Prompt](#5-advanced-prompt)
  - [5.1 basic prompt的缺陷](#51-basic-prompt的缺陷)
  - [5.2 Advanced Prompt技巧](#52-advanced-prompt技巧)
  - [5.3 Using temperature to vary your output](#53-using-temperature-to-vary-your-output)
- [6. text-generation-apps](#6-text-generation-apps)
- [7. building-chat-applications](#7-building-chat-applications)
  - [7.1 Integrating Generative AI into Chat Applications](#71-integrating-generative-ai-into-chat-applications)


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

#### 4.3.5 Supporting Content<br>
以某种形式影响`LLM输出`的内容被叫做`Supporting Content`,它可以是调整参数，格式化输出，主题分类等等<br>

#### 4.3.4 总结<br>
其实`prompt = instruction + Primary Content + Supporting Content`<br>
不一定全要加，用就完事了。<br>
给个用上的例子:<br>
```python
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()
client = OpenAI()

completion = client.chat.completions.create(
  model="gpt-3.5-turbo",
  messages=[
    {"role": "system", "content": "You are a sarcastic assistant."},  # 背景设定，算作complex的一环
    {"role": "user", "content": "请用诙谐的、嘲讽的风格来回答问题，每个问题回答一个词语即可"}, # instruction
    {"role": "user", "content": "xxx => 傻子"}, # primary_content: example
    {"role": "user", "content": "otto => 小丑"},# primary_content: example
    {"role": "user", "content": "请回答下列的问题"},# primary_content: cue
    {"role": "user", "content": "中国队 => "}, 
  ]
)

print(completion.choices)
# [Choice(finish_reason='stop', index=0, logprobs=None, message=ChatCompletionMessage(content='得意', role='assistant', function_call=None, tool_calls=None))]
```

## 5. Advanced Prompt<br>
第四节的`Prompt`只是小case，现在开始上强度教你用各种各样有趣的办法来提高`prompt水平`喽<br>

### 5.1 basic prompt的缺陷<br>
比如问题:`Generate 10 questions on geography.`<br>
看起来很大，却有2个大问题：<br>
```
1. 话题太大，地理相关内容可太多了
2. 格式问题，它不知道以何种格式输出
```

### 5.2 Advanced Prompt技巧<br>
```
1. Zero-shot prompting 
this is the most basic form of prompting. It's a single prompt requesting a response from the LLM based solely on its training data.

2. Few-shot prompting
this type of prompting guides the LLM by providing 1 or more examples it can rely on to generate its response.

3. Chain-of-thought
this type of prompting tells the LLM how to break down a problem into steps.

4. Generated knowledge
to improve the response of a prompt, you can provide generated facts or knowledge additionally to your prompt.

5. Least to most
like chain-of-thought, this technique is about breaking down a problem into a series of steps and then ask these steps to be performed in order.

6. Self-refine
this technique is about critiquing the LLM's output and then asking it to improve.

7. Maieutic prompting
What you want here is to ensure the LLM answer is correct and you ask it to explain various parts of the answer. This is a form of self-refine.
``` 
emm，其实对于`prompt`的技术分类也不是很明确就是了，你一定要说的话，**我觉得Chain of Thought其实也算few-shot的一种，Chain of Thought和 Least to most好像有差不多，都是在提供LLM分析问题/解决问题的例子**<br>
anyway，用就完事了~<br>

### 5.3 Using temperature to vary your output<br>
**PS: 在api中，可以通过temperature参数来确定答案的随机程度，从0到1，随机性越强**<br>


## 6. text-generation-apps<br>
开始实操一下，其实和之前写得代码差不太多<br>
```python
from openai import OpenAI
import os
import dotenv

# import dotenv
dotenv.load_dotenv()

# configure OpenAI service client 
client = OpenAI()

#deployment=os.environ['OPENAI_DEPLOYMENT']
deployment="gpt-3.5-turbo"

# add your completion code
no_recipes = input("No of recipes (for example, 5: ")
ingredients = input("List of ingredients (for example, chicken, potatoes, and carrots: ")
filter = input("Filter (for example, vegetarian, vegan, or gluten-free: ")

prompt = f"Show me {no_recipes} recipes for a dish with the following ingredients: {ingredients}. Per recipe, list all the ingredients used, no {filter}"
messages = [{"role": "user", "content": prompt}]  
# make completion
completion = client.chat.completions.create(model=deployment, messages=messages)

# print response
print(completion.choices[0].message.content)


old_prompt_result = completion.choices[0].message.content
prompt = "Produce a shopping list for the generated recipes and please don't include ingredients that I already have."

new_prompt = f"{old_prompt_result} {prompt}"
messages = [{"role": "user", "content": new_prompt}]
completion = client.chat.completions.create(model=deployment, messages=messages, max_tokens=1200,temperature=0.5)

# print response
print("Shopping list:")
print(completion.choices[0].message.content)

```

## 7. building-chat-applications<br>
**为了构建一个好用的chat application，首先需要回答如下2个问题**：<br>
```
1. 构建app： 我们如何针对特定用例高效构建和无缝集成这些人工智能驱动的应用程序？

2. 监控: 部署后，我们如何监控并确保应用程序在功能方面和遵守负责任人工智能的六项原则，能够以最高质量水平运行？
```

### 7.1 Integrating Generative AI into Chat Applications<br>
我们首先需要了解`chatbot`和`Generative AI-Powered Chat Application`的区别:<br>

|Chatbot|Generative AI-Powered Chat Application|
|-|-|
|专注于任务，基于规则|上下文感知|
|通常集成在一个大系统中|可以持有一个或多个chatbots|
|限制于编程功能|包含generative ai models|
|专业且结构化的交互|能够进行开放领域的讨论|

总之就是各种吹**Generative AI集成的聊天应用**啦<br>

为了更方便的开发ai应用程序，使用现成的先进**SDKs and APIs**是很重要的，好处如下:<br>
```
1. Expedites the development process and reduces overhead:
加速开发步骤并减少开销，很直观的原因，让你专注于逻辑

2. Better performance：
不用考虑程序的可拓展性或者突发的用户浏览徒增的问题，SDK和API已经提供了现成的解决方案

3. Easier maintenance：
更新只需要更换library即可

4. Access to cutting edge technology: 
利用在广泛的数据集上经过微调和训练的模型，为您的应用程序提供自然语言功能
```

