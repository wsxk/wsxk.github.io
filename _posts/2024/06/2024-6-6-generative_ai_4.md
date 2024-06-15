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
  - [11.3 Use Cases for using function calls](#113-use-cases-for-using-function-calls)
  - [11.4 Integrating Function Calls into an Application](#114-integrating-function-calls-into-an-application)
- [12. Designing UX for AI Applications](#12-designing-ux-for-ai-applications)


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

### 11.3 Use Cases for using function calls<br>
总之，使用所谓的`function calling`还有其他好处（有时候我真的挺佩服老美的，能想这么多奇葩好处<br>

```
Calling External Tools. 使用拓展工具 
Chatbots are great at providing answers to questions from users. By using function calling, the chatbots can use messages from users to complete certain tasks. For example, a student can ask the chatbot to "Send email to my instructor saying I need more assistance with this subject". This can make a function call to send_email(to: string, body: string)

Create API or Database Queries.  创建API或者数据库查询
Users can find information using natural language that gets converted into a formatted query or API request. An example of this could be a teacher who requests "Who are the students that completed the last assignment" which could call a function named get_completed(student_name: string, assignment: int, current_status: string)

Creating Structured Data.  创建结构化数据
Users can take a block of text or CSV and use the LLM to extract important information from it. For example, a student can convert a Wikipedia article about peace agreements to create AI flash cards. This can be done by using a function called get_important_facts(agreement_name: string, date_signed: string, parties_involved: list)
```

创建`function call`有3个主要步骤<br>
```
1、Calling the Chat Completions API with a list of your functions and a user message.

2、Reading the model's response to perform an action ie execute a function or API Call.

3、Making another call to Chat Completions API with the response from your function to use that information to create a response to the user.

```
如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240612230158.png)
小小的举个例子：<br>
```python
import os
import json
from openai import OpenAI
from dotenv import load_dotenv
load_dotenv()

client = OpenAI()
deployment = "gpt-3.5-turbo"

# step 1 : creating messages
messages= [ {"role": "user", "content": "Find me a good course for a beginner student to learn Azure."} ]
# step 2 : creating functions
functions = [
   {
      "name":"search_courses",
      "description":"Retrieves courses from the search index based on the parameters provided",
      "parameters":{
         "type":"object",
         "properties":{
            "role":{
               "type":"string",
               "description":"The role of the learner (i.e. developer, data scientist, student, etc.)"
            },
            "product":{
               "type":"string",
               "description":"The product that the lesson is covering (i.e. Azure, Power BI, etc.)"
            },
            "level":{
               "type":"string",
               "description":"The level of experience the learner has prior to taking the course (i.e. beginner, intermediate, advanced)"
            }
         },
         "required":[
            "role"
         ]
      }
   }
]
# step 3 : Making the function call
response = client.chat.completions.create(model=deployment,
                                        messages=messages,
                                        functions=functions,
                                        function_call="auto")

print(response.choices[0].message)
```
个人感觉**creating functions**这个步骤很精髓啊<br>
- `name` - The name of the function that we want to have called.
- `description` - This is the description of how the function works. Here it's important to be specific and clear.
- `parameters` - A list of values and format that you want the model to produce in its response. The parameters array consists of items where item have the following properties:
  1.  `type` - The data type of the properties will be stored in.
  1.  `properties` - List of the specific values that the model will use for its response
      1. `name` - The key is the name of the property that the model will use in its formatted response, for example, `product`.
      1. `type` - The data type of this property, for example, `string`.
      1. `description` - Description of the specific property.

There's also an optional property `required` - required property for the function call to be completed.

### 11.4 Integrating Function Calls into an Application<br>
直接看实例<br>
**这个实例告诉我们如何让LLM和function call联动起来，十分高级**<br>

```python
import os
import json
import requests
from openai import OpenAI
from dotenv import load_dotenv
load_dotenv()

client = OpenAI()
deployment = "gpt-3.5-turbo"

# step 1 : creating messages
messages= [ {"role": "user", "content": "Find me a good course for a beginner student to learn Azure."} ]
# step 2 : creating functions
functions = [
   {
      "name":"search_courses",   # 函数名称
      "description":"Retrieves courses from the search index based on the parameters provided", # 函数功能描述，简介准确
      "parameters":{   # 函数参数
         "type":"object", # 参数的类型，为object，即字典
         "properties":{   # 参数的属性有哪些，即字典中包含哪些属性
            "role":{     # 属性名称
               "type":"string", # 属性类型
               "description":"The role of the learner (i.e. developer, data scientist, student, etc.)" # 属性描述
            },
            "product":{
               "type":"string",
               "description":"The product that the lesson is covering (i.e. Azure, Power BI, etc.)"
            },
            "level":{
               "type":"string",
               "description":"The level of experience the learner has prior to taking the course (i.e. beginner, intermediate, advanced)"
            }
         },
         "required":[  # 必须参数，即参数中必须包含的属性名称
            "role"
         ]
      }
   }
]
# step 3 : Making the function call
response = client.chat.completions.create(model=deployment,
                                        messages=messages,
                                        functions=functions,  # 把函数描述放入这里，对于LLM来说，这就是可调用的函数列表
                                        function_call="auto") # 让LLM决定何时调用函数
print("response_message:")
print(response.choices[0].message)
print()

response_message = response.choices[0].message

def search_courses(role, product, level):   # 函数的实际实现
   url = "https://learn.microsoft.com/api/catalog/"   # 从目标api获取数据
   params = {
      "role": role,
      "product": product,
      "level": level
   }
   response = requests.get(url, params=params)
   modules = response.json()["modules"]
   results = []
   for module in modules[:5]:
      title = module["title"]
      url = module["url"]
      results.append({"title": title, "url": url})
   return str(results)

# Check if the model wants to call a function
if response_message.function_call.name:  # 这里在返回值会告诉你要调用函数
   print("Recommended Function call:")
   print(response_message.function_call.name)  # 打印要调用的函数名称
   print()

   # Call the function.
   function_name = response_message.function_call.name

   available_functions = {
            "search_courses": search_courses,
   }
   function_to_call = available_functions[function_name]  

   function_args = json.loads(response_message.function_call.arguments)
   print("function_args:")
   print(function_args)
   print()
   function_response = function_to_call(**function_args) # 实际根据函数名称，调用了真的函数，**在这里的作用是将一个字典展开为关键字参数（keyword arguments）

   print("Output of function call:")
   print(function_response)
   print(type(function_response))
   print()


   # Add the assistant response and function response to the messages
   messages.append( # adding assistant response to messages
      {
            "role": response_message.role,
            "function_call": {
               "name": function_name,
               "arguments": response_message.function_call.arguments,
            },
            "content": None
      }
   )
   messages.append( # adding function response to messages
      {
            "role": "function",
            "name": function_name,
            "content":function_response,
      }
   )

print("Messages in next request:")
print(messages)
print()

second_response = client.chat.completions.create(
   messages=messages,
   model=deployment,
   function_call="auto",
   functions=functions,
   temperature=0
      )  # get a new response from GPT where it can see the function response

print(second_response.choices[0].message.content)
```

## 12. Designing UX for AI Applications<br>
本节可以简单概述为，**基于用户体验和用户需要，基于合作和反馈机制，构建可信透明的ai应用。**<br>
