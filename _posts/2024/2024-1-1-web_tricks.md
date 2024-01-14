---
layout: post
tags: [web]
title: "web tricks"
date: 2024-1-1
author: wsxk
comments: true
---

- [前言](#前言)
- [0. web起手式三件套](#0-web起手式三件套)
- [1. view-source协议](#1-view-source协议)
- [2. 文件泄露](#2-文件泄露)
- [3. url编解码](#3-url编解码)
- [4. 域名](#4-域名)
- [5. rotots协议](#5-rotots协议)
- [6. 响应码](#6-响应码)
- [7. php探针](#7-php探针)


## 前言<br>
做简单的web 题目时，碰到一些有趣的trick，记录下来<br>

## 0. web起手式三件套<br>
F12查看、burpsuite抓包、dirsearch扫目录<br>

## 1. view-source协议<br>
`view-source协议是一种可以查看网页源代码的协议，可以通过在浏览器地址栏中输入view-source:网址来查看网页源代码，或者按下快捷键Ctrl+U来查看网页源代码。`<br>

## 2. 文件泄露<br>
很多情况是因为部署时，或者操作时出现了错误没有及时恢复<br>
常见的泄露方式有：<br>
```
1. .phps
2. .git
3. .svn
4. .swp //vim修改出问题导致的
5. .sql //如果开始网址大意了，就会漏sql的文件
6. 有的人不小心会把源码通过注释写到前端去，还不删除
```

## 3. url编解码<br>
https://www.cnblogs.com/liuhongfeng/p/5006341.html<br>

## 4. 域名<br>
查询域名解析地址 基本格式：nslookup host [server]<br>
查询域名的指定解析类型的解析记录 基本格式：nslookup -type=type host [server]<br>
查询全部 基本格式：nslookup -query=any host [server]<br>

## 5. rotots协议<br>
用于限定爬虫可以/不可以 爬取哪些页面的协议<br>
通常在网站的根目录下会有一个robots.txt文件<br>

## 6. 响应码<br>
301：永久重定向<br>
401：未授权<br>
403：禁止访问<br>
200：正常访问<br>

## 7. php探针<br>
PHP探针是用来探测空间、服务器运行状况和PHP信息的。探针可以实时查看服务器硬盘资源、内存占用、网卡流量、系统负载、服务器时间等信息。<br>
**探针用法：将探针的 PHP 文件上传到服务器的网站目录下，通过浏览器访问此 PHP 文件即可。**<br>
探针的默认名称为`tz.php`<br>
