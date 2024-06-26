---
layout: post
tags: [web]
date: 2024-5-13
title: "http协议&常见的http请求命令用法"
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
  - [2.1 get请求常见方法](#21-get请求常见方法)
  - [2.2 post请求常见方法](#22-post请求常见方法)
  - [2.3 redirect](#23-redirect)
  - [2.4 cookie](#24-cookie)
- [3. 网络命令的其他用途](#3-网络命令的其他用途)


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
### 2.1 get请求常见方法<br>
```
curl http://127.0.0.1:80 //向127.0.0.1:80发送http GET请求
curl -H "host: b11ab65a32592f0fd37ba7759a6de76f" http://127.0.0.1:80  //设置http header中的 host值为...
curl 127.0.0.1:80/fc3955bdfea3b9c7dac7d59e34185456 //设置path
curl http://127.0.0.1:80/a7ab7b05%206a9d8202/d92f5dee%20fcd0cf26 //%20为url encoding
curl http://127.0.0.1:80?a=dacd37221eb9ef75e5d50b24f296b17b
curl "http://127.0.0.1:80?a=607d2f069bcb2b86fcea941751f24fe7&b=1665f68d%20e6bbc013%26415f68e5%23a305dda1"

nc 127.0.0.1 80 
GET / HTTP/1.0 //进入nc后，输入这条请求，然后回车2次


nc 127.0.0.1 80
GET / HTTP/1.0
host: 100dcacc5ffc67b48792a2b30d7ee143

nc 127.0.0.1 80
GET /aef0ee14a27ef83e38b22b09180a103b HTTP/1.0

nc 127.0.0.1 80
GET /f1dd49e5%206aeed482/d8042f4e%20c2975dad HTTP/1.0

nc 127.0.0.1 80
GET /?a=0837188035ef848ed0806ac863d9fb46 HTTP/1.0

nc 127.0.0.1 80
GET /?a=52f494087c910cc921786b605d7234a8&b=d0c7acd4%20e16da5a2%263e4dc2b2%23634cea49 HTTP/1.0
```

```python
'''如何发送http get请求并设置格式'''
import requests
# url
url="http://127.0.0.1:80"
path="/b82e99c8%2092e052fc/573c4bf0%20795c1a4b"
query="?a=ac0faf4ebb8e0cd7160b88b2c95fe1c0&b=f4a8c59d%2057d882e5%2619341cea%23758df232"
url = url + path + query

# http header
headers = {
    'host': "c4e345405f816f210563e3e14d3977f3"
}

# send request
response = requests.get(url,headers=headers)

# print result
print(response.status_code)
print(response.text)
```

### 2.2 post请求常见方法<br>
```
curl -X POST -d "a=36bc958a729a642c1e439fa724628477" 127.0.0.1:80 
curl -X POST -d "a=952f68a808decdb76c6ff9d06044496e&b=92d2589b%205a1d8061%267dfd0805%256234d00" 127.0.0.1 80
curl -X POST -H "Content-Type: application/json" -d '{"a":"831bef1a7bb83227c0007e07138f3119"}' 127.0.0.1 80 //设置请求头为POST 且设置header中包含json格式， 并发送数据
curl -X POST -H "Content-Type: application/json" -d '{"a":"502882929ae954b6679eb27c2eb3fc0d", "b": {"c": "7f62289d", "d":["e1962678","fca36192 b740bc29&a6b562e7#49bd0dd7"]}}' 127.0.0.1 80   //curl 发送post请求，格式为json，附带很多数据


//cat request.txt
POST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 34

a=74b2d8fe7c4063c71f4b4235af364ea6
// shell 
cat request.txt | nc 127.0.0.1 80

//cat request.txt
POST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 78

a=165333bc6f3693dcd5666114b34a59ff&b=6f604c8d%208fdf70a4%26119c2577%23aff4f336
//shell
cat request.txt | nc 127.0.0.1 80

//cat request.txt
POST / HTTP/1.1
Content-Type: application/json
Content-length: 40

{"a":"7fe0bbfe024a48fe556f3e6b48bf31fa"}
//shell
cat request.txt | nc 127.0.0.1 80

//cat requests.txt
POST / HTTP/1.1
Content-Type: application/json
Content-length: 116

{"a":"1f505e240843ee9354938de975eec33f","b":{"c":"454b9523","d":["68459370","891941a7 7f3127e0&4117cd71#0b00fc7f"]}}
//shell
cat requests.txt | nc 127.0.0.1 80

```
`x-www-form-urlencoded`方法。<br>
```python
import requests
# url
url="http://127.0.0.1:80"

# data
data = {"a":"86ee51c2362aa07e4b66c91214b4b603","b":"68c23f54 e8131336&6bea23a9#c051674f"}

# send request
response = requests.post(url,data=data)

# print result
print(response.status_code)
print(response.text)
```

json格式发送报文:<br>
```python
import requests
# url
url="http://127.0.0.1:80"

#header
headers = {"Content-Type":"application/json"}

# data
data = {"a":"6f0bd5e837421e735dfab604004ea550","b":{"c":"81020930","d":["51f7240a","a709924d 7f163fe0&e51e575d#cf74727d"]}}

# send request
response = requests.post(url,json=data,headers=headers)

# print result
print(response.status_code)
print(response.text)
```

### 2.3 redirect<br>
有时候需要你重定向访问某个网址<br>
```
curl -L 127.0.0.1 80 //-L让curl自动跟随重定向网址

//nc的重定向需要知道重定向路径,随后再nc一次即可
```
```python
import requests
# url
url="http://127.0.0.1:80"

#header
headers = {"Content-Type":"application/json"}

# data
data = {"a":"6f0bd5e837421e735dfab604004ea550","b":{"c":"81020930","d":["51f7240a","a709924d 7f163fe0&e51e575d#cf74727d"]}}

# send request
response = requests.post(url,json=data,headers=headers,allow_redirects=True)

# print result
print(response.status_code)
print(response.text)
```

### 2.4 cookie<br>
如何用之前提到的命令带上cookie呢？<br>
```
// -L自动追踪重定向， -c 将服务器的cookie保存在cookie.txt中，-b会在每次访问时自动代入cookie.txt 
curl -L -c cookie.txt -b cookie.txt 127.0.0.1:80 


//cat request.txt 
POST / HTTP/1.1
Cookie: cookie=6b2bd093f4b0c05e75fa321ebaf4eed9
Content-Type: application/json
Content-length: 116

{"a":"1f505e240843ee9354938de975eec33f","b":{"c":"454b9523","d":["68459370","891941a7 7f3127e0&4117cd71#0b00fc7f"]}}
// 需要注意的点在于， nc不能像curl那样一手包办，需要先访问一次获得cookie，再把cookie带上
cat request.txt | nc 127.0.0.1 80
```
python 就很简单了，自动带cookie。<br>
```python
import requests
# url
url="http://127.0.0.1:80"

#header
headers = {"Content-Type":"application/json"}

# data
data = {"a":"6f0bd5e837421e735dfab604004ea550","b":{"c":"81020930","d":["51f7240a","a709924d 7f163fe0&e51e575d#cf74727d"]}}

# send request
response = requests.post(url,json=data,headers=headers,allow_redirects=True)

# print result
print(response.status_code)
print(response.text)
```


## 3. 网络命令的其他用途<br>
```
为了connect到 10.0.0.2 31337端口
nc 10.0.0.2 31337

为了监听本机的31337端口
nc -l 31337

扫描当前网段哪个ip开发了31337端口
namp -p 31337 10.0.0.0/24 

nmap -sS -p 31337 -T5 -Pn -n --min-rate 1000 10.0.0.0/16
-sS 使用TCP SYN扫描
-p 31337: 指定要扫描的端口为31337。
-T5: 设置扫描速度为最高（注意：这可能会增加被防火墙检测到的风险）。
-Pn: 跳过主机发现阶段，直接进行端口扫描。
-n: 禁用DNS解析以减少解析时间。
--min-rate 1000: 设定最小扫描速率为1000个包每秒。
10.0.0.0/16: 指定要扫描的网络范围为10.0.0.0到10.0.255.255。

tcpdump -A 'tcp and port 31337' -w capture.pcap
监控本地31337端口，监听tcp报文
-A 表示监控报文的数据并以ASCII形式打印
'tcp port 31337'表示监听tcp报文，端口是31337
-w表示把报文输出到文件中

tcpdump -A 'host 10.0.0.4 and host 10.0.0.2' -e
截获跟10.0.0.4与10.0.0.2相关的报文
-e告诉tcpdump显示以太网头信息，包括源和目的 MAC地址
```
