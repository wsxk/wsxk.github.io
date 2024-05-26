---
layout: post
tags: [AI]
title: "generative-ai 学习笔记 Ⅲ"
author: wsxk
comments: true
date: 2024-5-24
---

- [前言](#前言)
- [8. Building a Search Applications](#8-building-a-search-applications)
  - [8.1 what is semantic search](#81-what-is-semantic-search)
  - [8.2 what are Text Embeddings](#82-what-are-text-embeddings)
  - [8.3 How is the Embedding index created?](#83-how-is-the-embedding-index-created)
  - [8.4 how to search?](#84-how-to-search)

## 前言<br>
欢迎在阅读本篇之前，阅读[generative-ai 学习笔记 Ⅱ](https://wsxk.github.io/generative_ai_2/)<br>

## 8. Building a Search Applications<br>
本节中，我们会学习:<br>
```
1. Semantic vs Keyword search
了解语义搜索和关键词搜索的差别

2. What are Text Embeddings
Text Embeddings(vector) 是什么，有什么用

3. Creating a Text Embeddings Index. Searching a Text Embeddings Index.
如何创建和搜索 Text Embeddings
```

### 8.1 what is semantic search<br>
`semantic search`就是那种，你搜到`my dream car`，它会反应过来**我想要找的是ideal car**，而不会像`keyword search`那样，会遍历的搜寻**关于车的梦想**<br>

### 8.2 what are Text Embeddings<br>
`Text Embeddings`是一种把文本转换为向量（语义的数字表达）的技术<br>
下面是例子:<br>
```
Today we are going to learn about Azure Machine Learning.

// 实际上vector有很多，为了简便只列了10个
[-0.006655829958617687, 0.0026128944009542465, 0.008792596869170666, -0.02446001023054123, -0.008540431968867779, 0.022071078419685364, -0.010703742504119873, 0.003311325330287218, -0.011632772162556648, -0.02187200076878071, ...]
```

### 8.3 How is the Embedding index created?<br>
```
1. 将内容下载下来，比如 视频

2. 使用openai的功能，从前3分钟把视频的作者名称提取出来，放入vector database中

3. 将视频的文本切分成3mins的片段，为了保证片段的关联性，每个片段都包含和下一个片段重合的20个单词

4. 使用openai 的api功能，把每个文本片段让如并提取出60个词左右的总结，将总结放入vector database中

5. 最后，还是用openai的api，把每个文本变量的向量表达计算出来，放入vector database中
```
使用时，我们搜寻某个`内容`，这个`内容`会先被转化为`vector`，随后和数据库中的`vector`做比较，找到相似的，并将这个`vector`的文本总结、视频作者名称取出，就是我们想要的搜索内容啦<br>

### 8.4 how to search?<br>
上文提到的搜索某个东西,其实用到了**cosine similarity**的技术，在我们搜索的`内容`被转换成`vector`后，会与`vector database`的各个`vector`进行**cosine similarity**的计算来比对相似度。相似度高的会被取出。<br>

