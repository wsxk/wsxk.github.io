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
  - [11.2 Illustrating the problem through a scenario](#112-illustrating-the-problem-through-a-scenario)


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

### 11.2 Illustrating the problem through a scenario<br>
请看示例代码:<br>
```python
import os
import json
from openai import OpenAI
from dotenv import load_dotenv
load_dotenv()

client = OpenAI()
deployment = "gpt-3.5-turbo"

student_1_description="Emily Johnson is a sophomore majoring in computer science at Duke University. She has a 3.7 GPA. Emily is an active member of the university's Chess Club and Debate Team. She hopes to pursue a career in software engineering after graduating."
 
student_2_description = "Michael Lee is a sophomore majoring in computer science at Stanford University. He has a 3.8 GPA. Michael is known for his programming skills and is an active member of the university's Robotics Club. He hopes to pursue a career in artificial intelligence after finshing his studies."

prompt1 = f'''
Please extract the following information from the given text and return it as a JSON object:

name
major
school
grades
club

This is the body of text to extract the information from:
{student_1_description}
'''


prompt2 = f'''
Please extract the following information from the given text and return it as a JSON object:

name
major
school
grades
club

This is the body of text to extract the information from:
{student_2_description}
'''

response1 = client.chat.completions.create(
    model=deployment, 
    messages=[{"role": "user", "content": prompt1}]
    )
print(response1.choices[0].message.content)

response2 = client.chat.completions.create(
    model=deployment,
    messages=[{"role": "user", "content": prompt2}]
    )
print(response2.choices[0].message.content)
```

结果如下：<br>
```json
{
  "name": "Emily Johnson",
  "major": "computer science",
  "school": "Duke University",
  "grades": 3.7,
  "club": ["Chess Club", "Debate Team"]
}

{
  "name": "Michael Lee",
  "major": "computer science",
  "school": "Stanford University",
  "grades": "3.8 GPA",
  "club": "Robotics Club"
}
```
即使prompt是相同的，描述也很类似，但是grades这个变量的值并不是一样的。<br>
**这是因为LLM以写下的prompt的方式来提取非结构化的数据，并返回非结构化的数据。所以我们需要有一个结构化的格式，以便我们知道当存储或使用数据时，会发生什么**<br>
这就是要用`function calling`的原因。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240611230845.png)