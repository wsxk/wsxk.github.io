---
layout: post
title: "搭建属于自己的blog"
date:   2022-1-24
tags: [software_build]
comments: true
author: wsxk
---
好久以前我就有搭建博客记录自己所学的想法了。今天终于得偿所愿。
第一遍博客的内容当然是‘搭建自己的博客’啦。

##### 1.找到模板
我找到的模板如下：
[模板](https://github.com/lemonchann/lemonchann.github.io)

十分感谢这位师傅的模板！！
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-14-build_my_blog/1.png)

fork一个分支到自己的github 

##### 2.修改模板
fork一个分支后，里面有很多东西都是别人的，你需要修改文件内容。

一、_config.yml
直接搜关键词lemon，找到的可以根据注释进行修改。

二、about.md
这个是原作者的推广内容，也是需要该的（它会出现在你的博客里）

三、_posts目录（注意目录本身不能删除，所有博客的文档原件必须放到里面）
里面有一篇原作者的博客，需要删除

四、googledb06e025310e016e.html
没什么作用可以直接删

五、README.md
里面的内容是关于原作者的需要删除。

修改完毕后，直接网址上输入
username.github.io
能访问到里的网页，里面什么都没有

##### 3.gitalk
gitalk是评论区功能。开启步骤如下：
首先单击自己头像，点击settings
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-14-build_my_blog/2.png)

然后单击Developer settings
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-14-build_my_blog/3.png)

选择OAuth Apps，创建新的OAuth APP
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-01-14-build_my_blog/4.png)

创建完成后有Client ID
Client secrets 需要自己创建

然后复制两个到分支_config.yml中（搜索gitalk即可找到）修改对应信息即可。

