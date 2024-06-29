---
layout: post
tags: [AI]
title: "generative-ai 学习笔记 Ⅴ"
author: wsxk
comments: true
date: 2024-7-1
---

- [前言](#前言)


## 前言<br>
老规矩，请看完前面的章节，anyway其实你不看也没什么大问题~<br>
😄<br>

## 15. Retrieval Augmented Generation (RAG) and Vector Databases<br>
`RAG`主要解决的是**LLM训练数据集老旧，且去除了大量私人数据和公司产品手册的问题**<br>
本质逻辑是把最新材料作为`prompt`也放入用户输入中，作为上下文提供给LLM，让其给出合理答案。<br>
使用rag的好处如下：<br>
- 1. 信息更丰富，提供最新数据
- 2. 利用经过验证的数据来减少伪造
- 3. 相比fine-tune更省钱

### 15.1 RAG运行步骤<br>
```
1. 知识库基础：
最新数据库/文档被切分成chunk，随后喂入embedding model转换为vectors。

2. 用户查询：
用户查询某个问题。

3. retrieval（检索）：
用户查询问题后，这个问题会被检索，从知识库中检索出相关信息，将相关信息作为上下文和用户查询的问题结合，发送给LLM

4. LLM生成：
LLM基于检索的数据和用户的查询进行分析，得到响应并返还给用户。
Augmented Generation: the LLM enhances its response based on the data retrieved. It allows the response generated to be not only based on pre-trained data but also relevant information from the added context. The retrieved data is used to augment the LLM's responses. The LLM then returns an answer to the user's question.
```
根据检索到的数据的使用不同，RAG又分成了2个变体：<br>
```
RAG-Sequence,RAG-Sequence 在生成答案时的步骤如下：
1. 检索阶段：首先从知识库中检索与输入查询相关的多个文档。
2. 生成阶段：基于检索到的每个文档，独立生成一个候选答案。
3. 最终答案：从所有候选答案中选择一个作为最终答案，或者通过某种集成方法（如投票）生成最终答案。

RAG-Token,RAG-Token 在生成答案时的步骤如下：
1. 检索阶段：同样地，从知识库中检索与输入查询相关的多个文档。
2. 生成阶段：与 RAG-Sequence 不同的是，RAG-Token 在生成答案时是逐词生成的。即在生成每个词时，都会考虑所有检索到的文档。
3. 最终答案：逐词生成整个答案，每个词的生成都综合考虑了检索到的所有文档的信息。
```
**RAG-Sequence：基于每个检索文档生成多个完整答案，然后选择或集成这些答案。**<br>
**RAG-Token：逐词生成答案，每个词的生成都利用了所有检索到的文档的信息。**<br>

接下来会介绍如何执行1/3步骤的实现<br>

### 15.2 创建知识库(向量库)<br>
与传统数据库不同，向量数据库是一种专门用于存储、管理和搜索`embedding vector`的数据库。它存储文档的数字表示。将数据分解为`embeddings`使我们的 AI 系统更容易理解和处理数据。<br>
我们将`embeddings`存储在向量数据库中，因为 LLM 接受的`token`数量是有限的。由于无法将整个`embeddings`传递给 LLM，因此我们需要将它们分解成块`chunk`，当用户提出问题时，最像问题的`embeddings`将与提示一起返回。分块还可以降低通过LLM的`token`数量的成本。<br>
一些流行的矢量数据库包括 Azure Cosmos DB、Clarifyai、Pinecone、Chromadb、ScaNN、Qdrant 和 DeepLake。<br>
分块的代码如下:<br>
```python
def split_text(text, max_length, min_length):
    words = text.split()
    chunks = []
    current_chunk = []

    for word in words:
        current_chunk.append(word)
        if len(' '.join(current_chunk)) < max_length and len(' '.join(current_chunk)) > min_length:
            chunks.append(' '.join(current_chunk))
            current_chunk = []

    # If the last chunk didn't reach the minimum length, add it anyway
    if current_chunk:
        chunks.append(' '.join(current_chunk))

    return chunks
```

### 15.3 检索<br>
检索的核心思路有4种<br>
```
1. Keyword search - used for text searches

2. Semantic search - uses the semantic meaning of words

3. Vector search - converts documents from text to vector representations using embedding models. 
Retrieval will be done by querying the documents whose vector representations are closest to the user question.

4. Hybrid - a combination of both keyword and vector search.
```
这里面的关键点在于如何**衡量向量相似度**，目前常用的方法有`余弦相似度、欧几里得距离、点积`<br>

为数据库的每个向量创建索引的方法如下:<br>
```python
from sklearn.neighbors import NearestNeighbors

embeddings = flattened_df['embeddings'].to_list()

# Create the search index
nbrs = NearestNeighbors(n_neighbors=5, algorithm='ball_tree').fit(embeddings)

# To query the index, you can use the kneighbors method
distances, indices = nbrs.kneighbors(embeddings)
```

查询数据库后，您可能需要按最相关的顺序对结果进行排序。
```python
# Find the most similar documents
distances, indices = nbrs.kneighbors([query_vector])

index = []
# Print the most similar documents
for i in range(3):
    index = indices[0][i]
    for index in indices[0]:
        print(flattened_df['chunks'].iloc[index])
        print(flattened_df['path'].iloc[index])
        print(flattened_df['distances'].iloc[index])
    else:
        print(f"Index {index} not found in DataFrame")
```

总结:<br>
```python
user_input = "what is a perceptron?"

def chatbot(user_input):
    # Convert the question to a query vector
    query_vector = create_embeddings(user_input)

    # Find the most similar documents
    distances, indices = nbrs.kneighbors([query_vector])

    # add documents to query  to provide context
    history = []
    for index in indices[0]:
        history.append(flattened_df['chunks'].iloc[index])

    # combine the history and the user input
    history.append(user_input)

    # create a message object
    messages=[
        {"role": "system", "content": "You are an AI assiatant that helps with AI questions."},
        {"role": "user", "content": history[-1]}
    ]

    # use chat completion to generate a response
    response = openai.chat.completions.create(
        model="gpt-4",
        temperature=0.7,
        max_tokens=800,
        messages=messages
    )

    return response.choices[0].message

chatbot(user_input)
```


## 16. open-source-models<br>
anyway，在huggingface上使用一些开源模型就对了，省钱又省事~<br>
[https://huggingface.co/chat](https://huggingface.co/chat)

## 17. ai-agents<br>
AI 代理是生成式 AI 领域中一个非常令人兴奋的领域。这种兴奋有时会带来术语及其应用的混淆。为了使事情简单化并涵盖大多数涉及 AI 代理的工具，我们将使用以下定义：<br>
**人工智能代理通过让大型语言模型 (LLM) 访问状态和工具来执行任务。**<br>
其大概原理图如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240629225719.png)
开源项目`LangChain agent`就是其中之一，其工作原理如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240629225555.png)