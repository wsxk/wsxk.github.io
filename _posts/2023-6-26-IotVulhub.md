---
layout: post
tags: [iot]
title: "IoT-vulhub 漏洞复现"
date: 2023-6-26
author: wsxk
comments: true
---

- [写在前面](#写在前面)
- [install](#install)
- [1. Vivotek CC8160 栈溢出漏洞](#1-vivotek-cc8160-栈溢出漏洞)
  - [install](#install-1)
  - [触发漏洞](#触发漏洞)
- [2. 华为 HG532 远程代码执行漏洞（CVE-2017-17215）](#2-华为-hg532-远程代码执行漏洞cve-2017-17215)
- [3. TP-Link](#3-tp-link)
  - [SR20 本地代码执行漏洞](#sr20-本地代码执行漏洞)
  - [WR841N 栈溢出漏洞（CVE-2020-8423）](#wr841n-栈溢出漏洞cve-2020-8423)
- [4. Netgear](#4-netgear)
  - [Netgear R8300 upnpd 远程代码执行漏洞(PSV-2020-0211)](#netgear-r8300-upnpd-远程代码执行漏洞psv-2020-0211)
  - [Netgear R9000 命令注入漏洞（CVE-2019-20760）](#netgear-r9000-命令注入漏洞cve-2019-20760)
- [总结](#总结)


## 写在前面<br>
~~作为一名物联网安全研究员，怎么能不复现一下漏洞呢~~<br>
然而，为了一个漏洞去买一个设备感觉还是有点那啥（倒不是说没必要，新手可能需要先试试手，不至于浪费💴）<br>
`IoT-vulhub`就是一个有用的物联网漏洞复现平台，其集成了`firmadyne`以及`binwalk`，十分适合练手~<br>
然而，这个工具终究只适合练手，还有一部分的环境复现有问题!!!<br>

## install<br>
建议使用`ubuntu20.04`进行安装，因为其他环境我也没装过~<br>
**第一步，安装pip,建议在root下安装**<br>
```shell
curl -s https://bootstrap.pypa.io/get-pip.py | python3
```
**第二步，安装docker**<br>
```shell
curl -s https://get.docker.com/ | sh
```
顺带一提，为了能够让用户态进行docker访问，你需要参照[https://wsxk.github.io/docker_install/](https://wsxk.github.io/docker_install/)的步骤进行。<br>
**第三步，启动docker环境以及安装docker-compose**<br>
```shell
systemctl start docker
python3 -m pip install docker-compose
```
值得注意的是，缺什么就安装什么，所有报错均可以百度搜索到~<br>
安装完环境后，即可开始漏洞复现工作。<br>

## 1. Vivotek CC8160 栈溢出漏洞<br>
Vivotek是一家知名的家庭摄像头企业<br>
### install<br>
跟随[https://github.com/VulnTotal-Team/IoT-vulhub/tree/master/VIVOTEK/remote_stack_overflow](https://github.com/VulnTotal-Team/IoT-vulhub/tree/master/VIVOTEK/remote_stack_overflow)搭建环境即可~我才用的是系统模拟的方法<br>
顺道一提，在根据步骤操作到`构建镜像`时会出现问题，具体原因是`docker.io`提供的网站并没有`firmianay/qemu-system:armel`这个镜像，因此你需要自己在本地安装。<br>
**IoT-vulhub-master/baseImage/qemu-system/armel/images目录下，有一个download.sh文件，把其中的下载链接全都改成https://file.erlkonig.tech/debian-armel/xxxx 即可**<br>
随后在上一级目录下使用 `docker build -t firmianay/qemu-system:armel .`在本地构建镜像即可。<br>
还是在`构建镜像`步骤，在新版本docker中，执行`system-emu`目录下的`dockerfile`时会出现问题。`COPY ./firmware/_*/_31* /root/firmware`会执行失败。**因为在新版docker中，会对文件名做检测，所以，建议在文件中找到你提取的带有_31的目录到本地下，同时将命令改成`COPY ./firmware/_31.extracted /root/firmware`即可。**<br>

### 触发漏洞<br>
**用docker起了一个ubuntu16虚拟机，在其中运行qemu-system-arm运行目标程序**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230627113059.png)


## 2. 华为 HG532 远程代码执行漏洞（CVE-2017-17215）<br>
首先还是搭配环境，步骤和第一个漏洞复现类似，不过多赘述<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230627162113.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230627162136.png)
可以看到漏洞成功被复现。<br>

## 3. TP-Link<br>
### SR20 本地代码执行漏洞<br>
TP-Link SR20 是一款支持 Zigbee 和 Z-Wave 物联网协议可以用来当控制中枢 Hub 的触屏 Wi-Fi 路由器，此远程代码执行漏洞允许用户在设备上以 root 权限执行任意命令，该漏洞存在于 TP-Link 设备调试协议(TP-Link Device Debug Protocol 英文简称 TDDP) 中，TDDP 是 TP-Link 申请了专利的调试协议，基于 UDP 运行在 1040 端口。<br>
TP-Link SR20 设备运行了 V1 版本的 TDDP 协议，V1 版本无需认证，只需往 SR20 设备的 UDP 1040 端口发送数据，且数据的第二字节为 0x31 时，SR20 设备会连接发送该请求设备的 TFTP 服务下载相应的文件并使用 LUA 解释器以 root 权限来执行，这就导致存在远程代码执行漏洞。<br>
熟悉了前面的操作后，这个复现就显得轻车熟路了。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230627221304.png)
详细分析在[https://bbs.kanxue.com/thread-263539.htm](https://bbs.kanxue.com/thread-263539.htm)<br>
写得很细致~~<br>

### WR841N 栈溢出漏洞（CVE-2020-8423）<br>
在搭建完环境跟之前提到的步骤类似，还是不过多赘述。<br>
在搭建完环境后，可以通过`SSH`进行socks代理**注意在虚拟机内使用，而不是docker内**<br>
```shell
    ssh -D 2345 root@127.0.0.1 -p 1234
```
然后在`firefox`浏览器中使用代理设置:<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628163214.png)<br>
随后，可以在虚拟机中登录`docker`的界面<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628163414.png)
密码是`admin:admin`，登录后用火狐浏览器获得`cookies`以及登录的`url`<br>
```shell
curl -H 'Cookie: Authorization=Basic%20YWRtaW46MjEyMzJmMjk3YTU3YTVhNzQzODk0YTBlNGE4MDFmYzM%3D' 'http://192.168.2.2/DHJSQKMAPYZXIIXB/userRpm/popupSiteSurveyRpm_AP.htm?mode=1000&curRegion=1000&chanWidth=100&channel=1000&ssid='$(python -c 'print("/%0A"*0x55 + "aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaaccaacdaaceaacfaacgaachaaciaacjaackaaclaacmaacnaac")')''
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628162908.png)
重新刷新后，网站无法登录<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628163240.png)
模拟界面也出现`SIGSEGV`的字样<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628163615.png)

## 4. Netgear<br>
Netgear是一家国际知名的网络设备制造商，总部位于美国加利福尼亚州的圣何塞。该公司成立于1996年，主要专注于为家庭、企业和服务提供商制造和销售网络硬件。Netgear的产品线包括路由器、交换机、网络存储设备（NAS），以及其他网络设备和配件。<br>
对于家庭用户，Netgear提供各种无线路由器，这些路由器通常具有高性能和易于使用的特点。对于企业，Netgear提供更高级的网络解决方案，包括交换机和存储设备，以满足大规模网络的需求。<br>
Netgear还在网络安全方面提供一些产品和服务，包括VPN解决方案和网络防火墙。<br>
总的来说，Netgear是一家在网络硬件领域具有广泛影响力和知名度的公司。<br>
### Netgear R8300 upnpd 远程代码执行漏洞(PSV-2020-0211)<br>
好家伙，又是你`upnp`!<br>
在发送了崩溃`POC`后<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628205250.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628214138.png)

### Netgear R9000 命令注入漏洞（CVE-2019-20760）<br>
`Netgear`还是比较友好的，保留了所有以往的固件版本~<br>
通过它我们可以下载有漏洞的版本<br>
[https://www.downloads.netgear.com/files/GDC/R9000/R9000-V1.0.4.26.zip](https://www.downloads.netgear.com/files/GDC/R9000/R9000-V1.0.4.26.zip)<br>
以及修复了漏洞的版本<br>
[https://www.downloads.netgear.com/files/GDC/R9000/R9000-V1.0.4.28.zip](https://www.downloads.netgear.com/files/GDC/R9000/R9000-V1.0.4.28.zip)<br>
根据漏洞报告的提示，问题出自`web处理程序中`,其有一个明显的特征就是，文件中存在字符串`Referer`.**`Referer`字符串是一个HTTP头字段，通常在浏览器或其他HTTP客户端发出请求时发送给web服务器。它包含了用户是从哪个页面通过点击链接或提交表单来访问当前页面的信息。换句话说，它告诉服务器用户是从哪个URL来的。**<br>
使用`grep -r Referer . `，搜索结果如下：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628215324.png)
可以看出`uhttpd`是我们要找寻的目标<br>
使用`diaphora`来进行二进制比对操作,这也是一个强力的`diff`工具，和`bindiff`一致<br>
这个神秘的工具在`IDA7.7`上似乎有点问题，在`IDA7.6`上可以正常运行。怪<br>
主要在`partial matched`上寻找<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628222447.png)
右键，使用`diff pseudo code`来分辨<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230628223543.png)
可以发现，由`system`变成了`dni_system`，推测改动在这里产生。<br>

现在开始复现漏洞<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230629112753.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/20230629112853.png)



## 总结<br>
其实对于挖洞的过程还是没什么了解，总而言之还是学了些`docker`的使用方式、linux下shell运用以及常见的路由器漏洞，也不算太亏~<br>
