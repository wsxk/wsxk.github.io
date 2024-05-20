---
layout: post
tags: [web]
date: 2024-5-13
title: "http协议&common commands"
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
  - [1.8 Cookie](#18-cookie)
    - [1.8.1 Session ID](#181-session-id)
- [2. 常见的请求命令](#2-常见的请求命令)


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


### 1.8 Cookie<br>
1.1节中提到，`http`是一种**无状态**的协议，这意味着什么呢？就是服务器并不了解你先前跟它进行交互了内容->意味着如果你登录了某个网站，但是网站服务器在你登录后就不认你了，约等于白登了，这可怎么办?<br>
解决的办法是**在http的header中新增一个属性来维持状态，对于服务端来说，服务端返回一个cookie在http的响应头中: Set-Cookie,客户端拿到这个属性后，在接下来的交互中都在http的请求头中添加: Cookie**<br>
比如客户端发送了一个请求，如下:<br>
```
POST /login HTTP/1.0
Host: account.example.com
Content-Length: 32
Content-Type: application/x-www-form-urlencoded

username=Connor&password=password
```

服务端收到一个响应，如下:<br>
```
HTTP/1.0 302 Moved Temporarily
Location: http://account.example.com/
Set-Cookie: authed=Connor
```

接下来客户端在请求的时候：<br>
```
GET / HTTP/1.0
Host: account.example.com
Cookie: authed=Connor
```

服务端收到请求后，返回:<br>
```
HTTP/1.0 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 40
Connection: close

<html><body>Hello, Connor!</body></html>
```

#### 1.8.1 Session ID<br>
用cookie的时候有一个问题啊，像**authed=Connor这样的属性，可以直接被改为authed=admin，导致获得服务端的管理员权限，这并不是好事**<br>
`Session ID`的出现缓解了这个问题:<br>
客户端发送请求时:<br>
```
POST /login HTTP/1.0
Host: account.example.com
Content-Length: 32
Content-Type: application/x-www-form-urlencoded

username=Connor&password=password
```

服务端接受请求并返回:<br>
```
HTTP/1.0 302 Moved Temporarily
Location: http://account.example.com/
Set-Cookie: session_id=A1B2C3D4
```
至于**session_id=A1B2C3D4 标识为 用户Connor这个情报，会存储在服务端的数据库里**<br>
接下来的步骤就大差不差了<br>
```
GET / HTTP/1.0
Host: account.example.com
Cookie: session_id=A1B2C3D4
```

```
HTTP/1.0 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 40
Connection: close

<html><body>Hello, Connor!</body></html>
```


## 2. 常见的请求命令<br>
也介绍一下发送web请求的常见命令有哪些:<br>

```
curl 127.0.0.1:80 //向127.0.0.1:80发送http请求

nc 127.0.0.1 80 
GET / HTTP/1.0 //进入nc后，输入这条请求，然后回车2次
```