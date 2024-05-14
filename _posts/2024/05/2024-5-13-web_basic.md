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


## 前言<br>
简单的学习一下web前置知识吧~<br>

## 1. http协议<br>
### 1.1 RFC 1945(http/1.0)<br>
`RFC 1945`是定义了 HTTP/1.0 协议的一个文件，全称是 **Hypertext Transfer Protocol -- HTTP/1.0**<br>
这份文件描述了现在常见的网络架构：`Server和Client`使用`http`交互的方法。**值得一提的是，当前网络中已经有了http1.1 http2 http3等新的协议，有兴趣的小伙伴可以自行查阅**，这里为了学习简单，拿`http1.0`作为学习资料。`http 1.0`是一种通用的、无状态的、面向对象的应用层协议，这里会简单介绍一下它的作用<br>

### 1.2 http1.0 请求/回复格式<br>
`http1.0`的主要交互方式如下:<br>
一个`http request`: GET / HTTP/1.0<br>
响应的`http response`: HTTP/1.0 200 OK<br>
一个`http request`的格式如下:<br>

| Method| SP(separator) | Request-URI | SP | HTTP-Version|CRLF(就是'\r\n')|
|-|-|-|-|-|-|
|Get|空格|/|空格|HTTP/1.0| \r\n|


一个`http response`的格式如下:<br>

| HTTP-Version| SP(separator) | Status Code | SP | Reason-Phase|CRLF(就是'\r\n')|
|-|-|-|-|-|-|
|HTTP/1.0|空格|200|空格|ok| \r\n|

### 1.3 http1.0 状态码<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240514214913.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240514214956.png)

### 1.4 Method<br>
`http1.0`中有3个请求:**GET POST HEAD**<br>
其中**GET**请求用于**从服务器获取资源到客户端**<br>
**POST**请求用于**将客户端的信息传到服务器**<br>
**HEAD**请求用于**本质上和GET一样从服务器获取资源，但是不会真的获取资源，只是获取资源的大小**<br>

