---
layout: post
tags: [web]
date: 2024-5-13
title: "http协议"
author: wsxk
comments: true
---

- [前言](#前言)
- [1. http协议](#1-http协议)
  - [1.1 RFC 1945(http/1.0)](#11-rfc-1945http10)
  - [1.2 http1.0 请求/回复格式](#12-http10-请求回复格式)
  - [1.3 http1.0 状态码](#13-http10-状态码)
  - [1.4 Method](#14-method)
  - [1.5 HTTP URL Scheme](#15-http-url-scheme)
  - [1.6 url encoding](#16-url-encoding)
  - [1.7 Content-Type](#17-content-type)


## 前言<br>
简单的学习一下web前置知识吧~<br>
`PS: 这个章节只是记录一下各个web知识中的较为重要的记忆点，不会对web知识使用的整个流程做分析，并教导如何使用`<br>

## 1. http协议<br>
### 1.1 RFC 1945(http/1.0)<br>
`RFC 1945`是定义了 HTTP/1.0 协议的一个文件，全称是 **Hypertext Transfer Protocol -- HTTP/1.0**<br>
这份文件描述了现在常见的网络架构：`Server和Client`使用`http`交互的方法。**值得一提的是，当前网络中已经有了http1.1 http2 http3等新的协议，有兴趣的小伙伴可以自行查阅**，这里为了学习简单，拿`http1.0`作为学习资料。`http 1.0`是一种通用的、无状态的、面向对象的应用层协议，这里会简单介绍一下它的作用<br>

### 1.2 http1.0 请求/回复格式<br>
`http1.0`的主要交互方式如下:<br>
一个`http request`: GET / HTTP/1.0<br>
响应的`http response`: HTTP/1.0 200 OK<br>
一个`http request`的格式如下:<br>

| Method| SP(space) | Request-URI | SP | HTTP-Version|CRLF(就是'\r\n')|
|-|-|-|-|-|-|
|Get|空格|/|空格|HTTP/1.0| \r\n|


一个`http response`的格式如下:<br>

| HTTP-Version| SP(space) | Status Code | SP | Reason-Phase|CRLF(就是'\r\n')|
|-|-|-|-|-|-|
|HTTP/1.0|空格|200|空格|ok| \r\n|

### 1.3 http1.0 状态码<br>
http在返回响应时，会附加状态码，告诉你响应的信息<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240514214913.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240514214956.png)

### 1.4 Method<br>
`http1.0`中有3个请求:**GET POST HEAD**<br>
其中**GET**请求用于**从服务器获取资源到客户端**<br>
**POST**请求用于**将客户端的信息传到服务器**<br>
**HEAD**请求用于**本质上和GET一样从服务器获取资源，但是不会真的获取资源，只是获取资源的大小**<br>

### 1.5 HTTP URL Scheme<br>
`HTTP URL`是发送`http请求`的目标资源所在的位置标识，其结构如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240515225953.png)
在这个例子中<br>
```
scheme: http
host: example.com
port: 80
path: cat.gif
query: width=256&height=256
fragment: t=2s 
```

### 1.6 url encoding<br>
因为输入**url**访问某个网址是，有些字符无法被打印出来，或者识别时有矛盾之处，一般通过`url encoding`的方式，即`%+hex`的形式<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240515232601.png)

### 1.7 Content-Type<br>
在发送`POST`请求时，请求体的格式，常用的有2种`x-www-form-urlencoded`和`JSON`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240515232836.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240515232901.png)