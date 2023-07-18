---
layout: post
tags : [AI]
title: "Parameter and Dataset"
date: 2023-7-17
author: wsxk
comments: true
---

- [前言](#前言)
- [1. Parameters](#1-parameters)
- [2. DataSet](#2-dataset)
- [3. Parameter/DataSet setting](#3-parameterdataset-setting)
- [4. Other Optimization](#4-other-optimization)

## 前言<br>
话说回来,其实直觉上来说，对于一个大模型而言，参数越多，数据集越多，效果往往越好，如下图所示<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718135043.png)
**参数越多，数据集大小越大，Loss越少，模型功能越好**<br>
然而，有人好奇，参数大小和数据集大小应该使用多少比较合适呢?<br>

## 1. Parameters<br>
后来人们发现，大模型是有顿悟时刻`Emergent Ability`的，在保证数据量不变的情况下，参数达到一定水准后，大模型突然就变强了<br>
然而具体在哪个点变强，是不确定的，这给开发大模型带来了一些困难,即在一开始性能不强，或者增长不明显的时候，难以说服投资人继续砸钱开发大模型<br>
然而，会发生这样的情况是有理由的,下图可以比较清晰的解释为什么大模型在某一时刻效果达到max<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718135734.png)<br>
下面一张图更直观地展示了问题<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718135850.png)
在这种情形下，中模型甚至表现的还不如小模型（随便乱猜）<br>
**这告诉我们，应该对模型的评价标准进行更细粒度得划分。**<br>


## 2. DataSet<br>
下面一张图可以直观表现出数据集大小的重要性<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718140305.png)<br>
在数据集达到一定程度，模型不仅知道了语言的语法规则，还能掌握一些物理知识<br>
下图是一张对数据集进行裁剪的流程图<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718140407.png)
**实验表明，一个良好的数据集也有助于帮助模型学习，如果一个数据集杂质较多，不利于模型的学习**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718140524.png)

## 3. Parameter/DataSet setting<br>
在资源有限的情况下，如何设置模型大小和数据集数量，以便模型能达到最好的效果呢?<br>
在经过各种实验后，得到结论<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718141024.png)<br>
这种设置和现在的大模型相去甚远<br>
到底哪个比较好呢，又有实验表明，还是下图中数据比较合理<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230718141156.png)


## 4. Other Optimization<br>
还有一些常用的优化方法，比如先前提到的`In-context learning、Instruction tuning、Human Teaching(supervised learning)、RL（Reinforcement learning)`等等，可以在占用资源较少（相比于训练大模型而言）的情况下，获得比较好的提升<br>
