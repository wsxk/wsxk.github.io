---
layout: post
tags: [AI]
title: "generative-ai 学习笔记 Ⅲ"
author: wsxk
comments: true
date: 2024-5-24
---

- [前言](#前言)
- [8. Building Search Applications](#8-building-search-applications)
  - [8.1 what is semantic search](#81-what-is-semantic-search)
  - [8.2 what are Text Embeddings](#82-what-are-text-embeddings)
  - [8.3 How is the Embedding index created?](#83-how-is-the-embedding-index-created)
  - [8.4 how to search?](#84-how-to-search)
  - [8.5 实例](#85-实例)
- [9. Building  Image Applications](#9-building--image-applications)
  - [9.1 What is DALL-E and Midjourney?](#91-what-is-dall-e-and-midjourney)
  - [插曲: tokenization和text embedding](#插曲-tokenization和text-embedding)
  - [9.2 How does DALL-E and Midjourney Work](#92-how-does-dall-e-and-midjourney-work)
  - [9.3 instance](#93-instance)

## 前言<br>
欢迎在阅读本篇之前，阅读[generative-ai 学习笔记 Ⅱ](https://wsxk.github.io/generative_ai_2/)<br>

## 8. Building Search Applications<br>
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

### 8.5 实例<br>
看了一个比较有趣的sample<br>
写法值得学习，很高级（😄<br>
```python
import os
import pandas as pd
import numpy as np
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("OPENAI_API_KEY","")
assert API_KEY, "ERROR: OpenAI Key is missing"

client = OpenAI(
    api_key=API_KEY
    )

model = 'text-embedding-ada-002'

SIMILARITIES_RESULTS_THRESHOLD = 0.75
DATASET_NAME = "./08-building-search-applications/embedding_index_3m.json"


def load_dataset(source: str) -> pd.core.frame.DataFrame:
    # Load the video session index
    pd_vectors = pd.read_json(source)
    # for col in pd_vectors:
    #     print(col,end=": ")
    #     print(pd_vectors[col][0])
    return pd_vectors.drop(columns=["text"], errors="ignore").fillna("") # 删除名为text的列，如果没有则忽略，缺省值用空字符串填充

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)) # cosθ = a·b / |a||b|

def get_videos(
    query: str, dataset: pd.core.frame.DataFrame, rows: int
) -> pd.core.frame.DataFrame:
    # create a copy of the dataset
    video_vectors = dataset.copy()

    # get the embeddings for the query    
    query_embeddings = client.embeddings.create(input=query, model=model).data[0].embedding

    # create a new column with the calculated similarity for each row
    video_vectors["similarity"] = video_vectors["ada_v2"].apply(
        lambda x: cosine_similarity(np.array(query_embeddings), np.array(x))
    )

    # filter the videos by similarity
    mask = video_vectors["similarity"] >= SIMILARITIES_RESULTS_THRESHOLD
    # print(mask)
    video_vectors = video_vectors[mask].copy()
    # print(video_vectors)

    # sort the videos by similarity
    video_vectors = video_vectors.sort_values(by="similarity", ascending=False).head(
        rows
    )

    # return the top rows
    return video_vectors.head(rows)

def display_results(videos: pd.core.frame.DataFrame, query: str):
    def _gen_yt_url(video_id: str, seconds: int) -> str:
        """convert time in format 00:00:00 to seconds"""
        return f"https://youtu.be/{video_id}?t={seconds}"

    print(f"\nVideos similar to '{query}':")
    for _, row in videos.iterrows():
        print("_________________")
        print(_)
        print(row)
        print("_________________")
        youtube_url = _gen_yt_url(row["videoId"], row["seconds"])
        print(f" - {row['title']}")
        print(f"   Summary: {' '.join(row['summary'].split()[:15])}...")
        print(f"   YouTube: {youtube_url}")
        print(f"   Similarity: {row['similarity']}")
        print(f"   Speakers: {row['speaker']}")


pd_vectors = load_dataset(DATASET_NAME)


# get user query from imput
while True:
    query = input("Enter a query: ")
    if query == "exit":
        break
    videos = get_videos(query, pd_vectors, 5)
    display_results(videos, query)
```


## 9. Building  Image Applications<br>
图像生成还蛮重要的，有很多的应用:<br>
> 1. Image editing and synthesis. 
> > 图像编辑和合成可以用到
> 2. Applied to a variety of industries. 
> > 各种各样的行业也能用到，例如：医疗科技、旅游、游戏开发

### 9.1 What is DALL-E and Midjourney?<br>
`DALL-E`和`Midjourney`是两款比较著名的图片生成大模型。<br>
它们都允许你用用户prompts生成图片<br>
```
DALL-E

由2个部分组成：CLIP和 Diffused attention

CLIP是生成embedding的模型，将用户输入（图片和文本）转换为embeddings

Diffused attention用于读入embeddings，输出图片
```

`Midjourney`与之类似，也能生成图片。<br>


### 插曲: tokenization和text embedding<br>
```
Tokenization是预处理阶段，用于将文本分解为基本单元。
Text Embeddings是特征提取阶段，用于将tokens转换为向量表示。
```

### 9.2 How does DALL-E and Midjourney Work<br>
对于`DALL-E`而言<br>
```
DALL-E 是一个 Generative AI model， 基于 transformer 架构，还带有一个 autoregressive transformer

autoregressive transformer定义了模型如何根据文本描述生成图像，它一次生成一个像素，然后使用生成的像素生成下一个像素。经过神经网络中的多层，直到图像完整

通过此过程，DALL-E 可以控制其生成的图像中的属性、对象、特征等。但是，DALL-E 2 和 3 对生成的图像的控制更强。
```

### 9.3 instance<br>
来点实例：<br>
```python
from openai import OpenAI
import os
import requests
from PIL import Image
import dotenv
from io import BytesIO

# import dotenv
dotenv.load_dotenv()

 
client = OpenAI()


try:
    # Create an image by using the image generation API
    generation_response = client.images.generate(
        model="dall-e-3",
        prompt='Bunny on horse, holding a lollipop, on a foggy meadow where it grows daffodils',    # Enter your prompt text here
        size='1024x1024',
        n=1,
    )
    # Set the directory for the stored image
    image_dir = os.path.join(os.curdir, 'images')
    
    # If the directory doesn't exist, create it
    if not os.path.isdir(image_dir):
        os.mkdir(image_dir)
    print(image_dir)

    # Initialize the image path (note the filetype should be png)
    image_path = os.path.join(image_dir, 'generated-image.png')
    print(image_path)

    # Retrieve the generated image
    print(generation_response)

    image_url = generation_response.data[0].url # extract image URL from response
    generated_image = requests.get(image_url).content  # download the image
    with open(image_path, "wb") as image_file:
        image_file.write(generated_image)

    # Display the image in the default image viewer
    image = Image.open(image_path)
    image.show()

# catch exceptions
except client.error.InvalidRequestError as err:
    print(err)

# ---creating variation below---
response = client.images.create_variation(
  image=open(image_path, "rb"),
  n=1,
  size="1024x1024",
)
print(response)
image_url_variation = response.data[0].url
print(image_url_variation)
generated_image_variation = requests.get(image_url_variation).content
img = Image.open(BytesIO(generated_image_variation))
img.show()
```