---
layout: post
tags: [software_build]
title: "GoldenDict导入词典"
date: 2022-8-11
author: wsxk
comments: true
---

GoldenDict是一款开源并且强大的词典查询ui工具。<br>
其作用十分简单:导入词典格式的文件方便查询<br>
推荐GoldenDict的原因也很简单，它支持最多的文件格式解析。<br>
稍微探索了一下软件后，大概知道了如何使用并导入词典。

## GoldenDict安装<br>
[https://github.com/goldendict/goldendict](https://github.com/goldendict/goldendict)<br>
为源码地址，但是我们肯定不用源码编译，太麻烦了，可以考虑源码编译后的压缩包，开袋即食:<br>
[https://github.com/goldendict/goldendict/wiki/Early-Access-Builds-for-Windows](https://github.com/goldendict/goldendict/wiki/Early-Access-Builds-for-Windows)
<br>
打开后，界面是长这样的:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220811152603.png)
此时可以按 **F3** 出现如下界面：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220811152646.png)
可以通过添加按钮添加词典文件所在的文件夹，这样GoldenDict就会自动识别到词典文件并解析。

词典文件通常由几个部分组成:
> 1. mdx文件：该类型文件包含了词典的所有 ***文本信息***
> 2. mdd文件：一个图文并茂的字典肯定不止只有文本内容，mdd包含如图片，语音，css文件，js脚本等
> 3. css文件：修饰页面，使界面更好看。