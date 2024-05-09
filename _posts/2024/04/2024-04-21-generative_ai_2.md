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
