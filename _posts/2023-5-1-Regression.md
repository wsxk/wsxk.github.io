---
layout: post
tags: [AI]
title: "AI Regression&Classification&Logistic Regression"
date: 2023-5-1
author: wsxk
comments: true
---

- [Regression](#regression)
  - [1. Model](#1-model)
  - [2. Goodness of Function](#2-goodness-of-function)
  - [3. Best Function](#3-best-function)
  - [举例:如果有2个参数](#举例如果有2个参数)
  - [结果如何](#结果如何)

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

## Regression<br>
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

### 结果如何<br>
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
