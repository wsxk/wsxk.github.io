---
layout: post
tags: [AI]
title: "AI Regression&Classification&Logistic Regression"
date: 2023-6-24
author: wsxk
comments: true
---

- [一、Regression](#一regression)
  - [1. Model](#1-model)
  - [2. Goodness of Function](#2-goodness-of-function)
  - [3. Best Function](#3-best-function)
  - [举例:如果有2个参数](#举例如果有2个参数)
  - [如何做的更好](#如何做的更好)
- [二、Classification: Probabilistic Generative Model](#二classification-probabilistic-generative-model)
  - [当成Regression问题?](#当成regression问题)
  - [如何处理](#如何处理)
  - [1. Model](#1-model-1)
    - [结果如何](#结果如何)
    - [优化：公用同一组covariance](#优化公用同一组covariance)
  - [2. Loss Function](#2-loss-function)
  - [3. Best Function](#3-best-function-1)
  - [Probability Distribution](#probability-distribution)

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

## 一、Regression<br>
**回归（Regression），指研究一组随机变量(Y1 ，Y2 ，…，Yi)和另一组(X1，X2，…，Xk)变量之间关系的统计分析方法，又称多重回归分析。通常Y1，Y2，…，Yi是因变量，X1、X2，…，Xk是自变量。**<br>
我们这里研究的是`线性回归（Linear Regression）`，说人话就是，**线性回归分析是根据一个或一组自变量（X1,X2,...Xk）的变动情况预测与其相关关系的某随机变量(Y)的未来值的一种方法**<br>
`线性分析是AI中经常运用到的技术`<br>
线性回归的例子也有很多，比如`预测股票市场(输入前10年的股票行情，输出当前股票的预测情况)，无人车驾驶(输入各种道路信息，输出方向盘旋转角度)、推荐系统（输入使用者和商品，输出购买的可能性）`<br>
以预测宝可梦的进化后战力为例子<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701152909.png)
总共可以分成3个步骤<br>
### 1. Model<br>
第一步是确定模型<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701152944.png)
假定一个模型<br>
$y = b + w*x_{cp}$

### 2. Goodness of Function<br>
第二步是判断函数的好坏<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701153618.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701153635.png)<br>
判断函数好坏有一个叫做 `Loss Function(输入是一个函数，输出一个函数的好坏)`的东西，其实是衡量一个函数是否优秀的标准，其实就是求方差<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701153730.png)

### 3. Best Function<br>
第三步是找到一个最好的函数<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701153838.png)
那么如何找到最好的那个函数呢<br>
这里使用了 **Gradient Descent(梯度下降)** 的方法<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701154036.png)
本质上是求斜率，**斜率是负数，说明w增加会让L(W)减少，所以要增加w；斜率是正数，说明w增加会让L(w)增加，所以要减少w**<br>
**然而，我们也要w一次需要减多少的问题**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701154303.png)
算法如下:
1. 随机选一个初始值 $w_{0}$<br>
2. 进行一次梯度下降（重复这个步骤，直到斜率为0）<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701154359.png)
这里有人可能担心，如果达到极值点（**Local optimal**）而不是最小值点（**global optimal**）怎么办<br>
**线性回归(Linear Regression)没有Local optimal**<br>

### 举例:如果有2个参数<br>
其实很类似，就是求斜率变成了求偏导值<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701154722.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701154738.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701154822.png)

### 如何做的更好<br>
**我们用训练数据训练了一个模型，我们真正关心的是这个模型在未来的预测结果精准度如何**<br>
如何做的更好?<br>
**1. 更复杂的模型**<br>
用更高次幂的函数来替代一开始的模型<br>
**然而需要注意的是，并不是模型越复杂越好，太复杂的模型容易造成过拟合(overfitting)(只在训练时效果好，在实际测试时差距很大)**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701155100.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701155232.png)

**2. 增加训练集，发现隐藏的因子**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701155354.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701155414.png)

**3. Regularization，重新定义更好的Loss Function**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701155454.png)
直觉上，w越小越好，因为这代表更加平滑。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230701155510.png)
**然而也不是越平滑越好，太平滑就是一个直线，这反而没什么意义**<br>
**做Regularization时不考虑bias，因为它只是移动函数位置，不影响平滑程度**<br>


## 二、Classification: Probabilistic Generative Model<br>
本文主要介绍classification分类，分类是指这样一个函数，输入一个对象x，经过函数之后的输出是这个对象属于哪个类别，比如医疗诊断，把一个病人的年纪性别症状等进行输入，它的输出是具体的哪一种病，比如信用卡借贷系统，把一个人的年纪，收入，职业等输入，输出是=的是系统判定的能否借钱。值得注意的是，在分类函式中，类别是事先已经存在的，只是针对某种输入，它的输出是已有类别中的某一个。
***分类问题通常和概率分布函数挂钩***<br>
咱们来看些例子:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706102256.png)
拿一个具体的例子来说，我们可以对宝可梦中的精灵进行属性分类。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706102431.png)
宝可梦有七种数据，`Total、HP、ATK、Defense、SP ATK、SP Defense、Speed`。<br>
为了做分类，第一步当然还是收集数据，即**宝可梦的数据和属性**<br>
### 当成Regression问题?<br>
有人会问，当成regression问题来做怎么样?<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706103123.png)
结果会导致如下情形:<br>
**1、结果太好反而导致得到的模型不适用**<br>
**2、在多种种类中，可能并没有相关性，设置成class 1（1），class 2（2），其实是说明class 1和class 2 是有关系的，然而实际上可能并没有关系**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706103609.png)

### 如何处理<br>
其实还是`function、Loss function、find best function`三步走。只不过Function不再是线性回归了<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706104010.png)

### 1. Model<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706104513.png)
把`BOX`换成`Class`后<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706104757.png)
实际上，从样本集当中训练的目的变成了，估算概率<br>
这一整套想法叫做**Generative Model**<br>
开始实际做一下<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706104904.png)
然后就出现了一个问题<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706110658.png)
如果一个精灵，没有出现在训练集当中，那么它的概率应该如何计算？如果是0，这就是一个垃圾模型，没有意义。<br>
实际上<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706111242.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706111327.png)
如果我们知道了这两个值`mean和 covariance`，就可以
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706111556.png)
现在的问题就是如何找`mean和covariance`<br>
这里用到了一个`Maximum Likelihood`的说法<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706112038.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706112154.png)

#### 结果如何<br>
那宝可梦水系和一般系举例<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706112314.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706112431.png)
算完后，得到结果<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706112647.png)
用二维，得到47正确率，然而用了七维，也只有54正确率....

#### 优化：公用同一组covariance<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706113217.png)
逻辑是，减少参数数量，防止`overfitting`<br>
那么公用的`covariance`是什么呢？<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706113426.png)
公用一组`covariance`后，得到结果<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706113838.png)

### 2. Loss Function<br>
这里的`Loss Function`可以是训练集上输出的错误次数，把不同的高斯分布的`Loss`都计算出来<br>
### 3. Best Function<br>
从上述结果里挑选一个`Loss`最小的`Function`即为所求<br>

### Probability Distribution<br>
为什么要用高斯分布模型？<br>
其实这个是人的智慧，**你可以选择你想要的几率分布模型**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230706115209.png)
