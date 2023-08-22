---
layout: post
tags: [iot_dev]
title: "iot dev: Project management"
author: wsxk
date: 2023-8-7
comments: true
---

- [1. project arch](#1-project-arch)
- [2. 嵌入式系统概论与开发流程](#2-嵌入式系统概论与开发流程)
  - [一、概论](#一概论)
  - [二、嵌入式系统开发项目生命周期](#二嵌入式系统开发项目生命周期)
- [3. 项目管理与软件工程](#3-项目管理与软件工程)
  - [一、嵌入式项目管理](#一嵌入式项目管理)


## 1. project arch<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-7-6/20230808155202.png)

## 2. 嵌入式系统概论与开发流程<br>
### 一、概论<br>
嵌入式系统的定义一般如下:**Embedded System use general or specialized purpose CPUsrunning custom softwarealong with specialized hardwareto perform applica-tion-specificfunctions.**<br>

在设计一个嵌入式应用项目时，通常需要考虑以下几点：<br>
1. 成本 
2. 外观　
3. 预计销售市场与消费群体
4. CPU计算能力
5. 内存大小
6. 省电需求
7. 稳定度
8. 反应实时性（Real-Time）
9. 软件复杂度
10. 测试复杂度

**其实某项产品是否符合嵌入式系统的定义并不重要，重要的是开发者必须完全了解嵌入式系统的本质，以避免在设计开发阶段做出错误的判断与决定**<br>

### 二、嵌入式系统开发项目生命周期<br>
**Schedule是用来遵守的，不是用来修改的**<br>
客户来说明嵌入式项目时，在跟开发企业说明时间预算时，总是偷偷自己预留了部分时间，说的时候总是要求时间ASAP（As soon as possible）<br>
客户在规格方面也会自己留下缓冲空间，在刚开始谈规格时偶尔会模糊其词,如果我们可以做到客户的高标准当然最好，假使不行的话，只要没有低于客户藏在心里的底限，其实都还是会被接受的(我超，都是心机啊)<br>
**质量是规划、设计出来的，不是检查出来的。**<br>
第3章：嵌入式系统开发项目生命周期：项目启动与规划■　
第4章：嵌入式系统开发项目生命周期：设计、执行与结项■　
第17章：系统整合■　
第18章：Testing、Debugging与Tuning■　
第19章：结项前的煎熬■　
附录D：电子产品设计的最终依据：用户体验


## 3. 项目管理与软件工程<br>
### 一、嵌入式项目管理<br>
**任何工作都应该先评估可行性，接着作计划，然后有效率地利用时间、成本与资源，并在可接受的范围内管控成果的质量**<br>
有一个关键思想`PDCA(plan-do-check-act)`,即谋定而后动，做完后要反思，争取下一次做得更好<br>
**WBS(Work Breakdown Structure)** 很重要<br>
在更改计划时，**严格禁止接受客户私下变更规格的请托。**<br>
　
第15章：项目进度追踪实务■　
第16章：SoC设计公司中嵌入式系统团队的管理■　
附录A：未执行项目管理的项目

