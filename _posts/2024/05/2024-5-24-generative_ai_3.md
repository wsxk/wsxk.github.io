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
