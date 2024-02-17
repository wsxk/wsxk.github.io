---
layout: post
tags: [software_build]
title: "Apache+PHP install on win11"
date: 2024-1-15 
author: wsxk
comments: true
---

- [前言](#前言)
- [1. Apache](#1-apache)
  - [1.1 下载安装包](#11-下载安装包)
  - [1.2 修改配置](#12-修改配置)
  - [1.3 启动](#13-启动)
- [2. PHP](#2-php)
  - [2.1 官网下载](#21-官网下载)
  - [2.2 配置apache](#22-配置apache)
  - [2.3 验证](#23-验证)


## 前言<br>
win11下安装Apache+PHP环境，以便于本地开发测试。以及做一些web题目<br>
**注意：因为把php、apache安装在c盘中，每一次修改配置文件都需要admin权限，同时，启动网络服务也需要admin权限**<br>

## 1. Apache<br>
### 1.1 下载安装包<br>
去Apache官网的时候看到Apache本身不构建windows安装包，但是提供了几个源，可以下载，其中就有[https://www.apachehaus.com/](https://www.apachehaus.com/)<br>
选择下图中的`Downloads`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240115213332.png)
有一个`x64`的标记<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240115213407.png)
下载即可<br>
### 1.2 修改配置<br>
打开目录中的`httpd.conf`<br>
将下图的`Define SRVROOT`改成自己apache24安装的位置即可<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240115213621.png)

### 1.3 启动<br>
将Apache的`bin`目录添加到环境变量中<br>
然后win terminal启动<br>
```
httpd -t #测试配置文件是否合法
httpd -k install -n Apache2.4 #-n后面表示自定义访问名称


httpd -k start  #启动apache
httpd -k stop   #停止apache 或者可以在windows中的service中启动或停止
```
启动后，打开浏览器，输入`localhost`即可看到apache的默认页面<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240115213841.png)

## 2. PHP<br>
### 2.1 官网下载<br>
[https://windows.php.net/download](https://windows.php.net/download)<br>
选择zip文件下载，下载完后解压，同样，将存在`php.exe`的目录添加到环境变量中<br>
### 2.2 配置apache<br>
打开`httpd.conf`<br>
```
#加载PHP
LoadModule php_module 'C:/Program Files/WebServer/php-8.3.1-Win32-vs16-x64/php8apache2_4.dll'

#将PHP配置文件加载到Apache配置文件中，共同生效
PHPIniDir 'C:\Program Files\WebServer\php-8.3.1-Win32-vs16-x64'

#配置Apache分配工作给PHP模块，把PHP代码交给PHP处理
#即.php后缀名的文件
AddType application/x-httpd-php .php
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240115214253.png)

**注意，上面php安装目录中的php.ini原本是不存在的，可以把`php.ini-development`复制一份，并重新命名php.ini**<br>
**完成上述步骤后，重启apache服务**<br>

### 2.3 验证<br>
随便写个php文件，放到apache的`htdocs`目录下，然后访问即可<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240115214547.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240115214642.png)
成功！<br>

