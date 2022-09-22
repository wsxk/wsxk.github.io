---
layout: post
title: "QQ/微信windows端防撤回"
date:   2022-2-10
tags: [RealWorld]
comments: true
author: wsxk
---

- [相关环境<br>](#相关环境)
- [QQ<br>](#qq)
    - [单独聊天撤回](#单独聊天撤回)
    - [群聊撤回](#群聊撤回)
    - [最终](#最终)
- [微信<br>](#微信)

起因是我跟我朋友聊天的时候，他撤回了东西，之后问他，他又不说，还卖关子，我特别生气
于是搜了一下QQ的防撤回

还真搜到了，关于qq防撤回的函数都在IM.dll的动态库中，我们只要修改了它的相关函数即可以让朋友闻风丧胆，看他们以后谁撤回！！！

捏妈妈滴！


## 相关环境<br>
IDA(带keypatch)

## QQ<br>
#### 单独聊天撤回
ida打开IM.dll
搜索字符串 "bytes_reserved"
可以看的两个一样的字符串
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/1.png)

可能大家会好奇为什么有两个一模一样的，根据我的猜测
这两个字符串
一个用于你撤回消息时
一个用于别人撤回消息时

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/2.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/3.png)

其中aBytesReserved_0为标签的是别人撤回消息时使用的字符串。

我们要做的是修改aBytesReserved_0为标签被使用的位置
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/4.png)

这里在push ecx的位置开始修改汇编，使其直接跳到test eax,eax的位置

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/5.png)

修改完毕即可

#### 群聊撤回
搜索"bytes_userdef"字符串
同样有2个
我们要修改的是aBytesUserdef_0标签被使用的位置
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/6.png)

但是注意，beytes_unserdef字符串被引用了2次，这2次都要修改
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/7.png)


2次的操作一样，汇编代码也类似，只说其中一个的修改

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/8.png)

从 push offset aBytesUserdef_0开始，修改，使其直接跳转到test eax，eax的位置。

修改后如下
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-2-10-QQ%E9%98%B2%E6%92%A4%E5%9B%9E/9.png)

对另外一个位置进行相同操作。


#### 最终
完成后保存到IM.DLL中
最后关闭QQ
把你修改后的IM.DLL覆盖QQ文件夹中原本的IM.DLL
重新打开后完成！

## 微信<br>
微信的撤回功能在WeChatWin.dll里<br>
ida打开，搜索字符串"revokemsg"<br>
重点是revokemsg前一个jz跳转指令改为jnz<br>