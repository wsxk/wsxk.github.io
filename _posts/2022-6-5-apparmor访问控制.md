---
layout: post
title: "AppArmor 访问控制"
date:   2022-6-5
tags: [linux]
comments: true
author: wsxk
---

- [简介](#简介)
- [安装](#安装)
- [apparmor基本操作](#apparmor基本操作)
  - [启动和关闭](#启动和关闭)
  - [对一个程序添加限制](#对一个程序添加限制)
- [apparmor使用](#apparmor使用)
  - [1.手动编写](#1手动编写)
  - [2.利用工具自动生成](#2利用工具自动生成)
- [apparmor模式](#apparmor模式)
  - [complain模式](#complain模式)
  - [enforcement](#enforcement)
- [apparmor控制细节](#apparmor控制细节)
  - [1.文件系统的访问控制](#1文件系统的访问控制)
  - [2.资源限制](#2资源限制)
  - [3.网络控制](#3网络控制)
  - [完整例子](#完整例子)
- [reference](#reference)

# 简介

AppArmor(Application Armor)是Linux内核的一个安全模块，AppArmor允许系统管理员将每个程序与一个安全配置文件关联，从而限制程序的功能

Apparmor在ubuntu上已经集成，可以限制已知应用的能力，控制应用访问文件、目录和网络的能力。具有类似功能的工具还包括selinux、lids，目前selinux、lids和apparmor遵循LSM框架，并通过LSM框架整合到内核中

# 安装

本人只在ubuntu上使用过此功能，其它linux发行版请自行搜索安装教程。

apparmor这项功能在ubuntu内核上其实是已经集成好了的，但是命令有限，操作起来不太方便，可以通过以下增添功能

    sudo apt-get install apparmor-utils

# apparmor基本操作

## 启动和关闭

    sudo systemctl enable apparmor //允许该服务运行
    sudo systemctl disable apparmor//禁止该服务运行

    sudo systemctl start apparmor.service   //开启和服务
    sudo systemctl stop apparmor.service   //停止服务

    sudo systemctl status apparmor.service   //服务状态
    sudo systemctl reload apparmor.service   //加载配置

## 对一个程序添加限制

在你编写完一个关于某程序的配置文件profile后，使用如下命令进行添加和删除

    apparmor_parser -r /path/to/profile #添加入内核中

    apparmor_parser -R /path/to/profile #从内核中移除

2个命令都需要你启动了apparmor.service后才能使用成功。

# apparmor使用

## 1.手动编写

举个栗子

下面的代码就是一个有关test程序的访问控制配置文件

    #include <tunables/global>

    /home/seed/Desktop/information_secure/experiment2/test { //程序目录
        #include <abstractions/apache2-common>
        #include <abstractions/base>
        #include <abstractions/dovecot-common>
        #include <abstractions/postfix-common>

        capability dac_override ,
        capability dac_read_search,
        
        /bin/bash rix,
        /bin/cat mrix,
        /bin/dash mrix,
        /bin/ls mrix,
        /dev/tty rw,
        /home/*/Desktop/information_secure/experiment2/ r,
        /home/*/Desktop/information_secure/experiment2/readme r,
        /home/seed/Desktop/information_secure/experiment2/test mr , 
        /proc/filesystems r,
        /usr/bin/whoami mrix,
        /usr/bin/ncat rix,
    }

写完后 通过

    apparmor_parser -r /path/to/profile

即可加载进内核

## 2.利用工具自动生成

理论上手动写就可以了，但是你写这么东西你自己不觉得麻烦么，特别是如果要对访问控制进行细粒度操控的话，要写的东西多得一批。

我们可以使用 aa-genprof命令针对某个可执行程序自动生成配置文件

用法 

    aa-genprof binary

运行后会出现该提示

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-5-apparmor%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6/20220605190102.png)

简单翻译，就是要你在另一个终端跑一遍这个程序，然后选scan system log 的选项，它会自动根据这个程序的行为（在system log里面查看)为你自定义选项

你之后可以设置允许或者其他。 完成后，可以选择finish选项。

该命令生成的配置文件在 /etc/apparmor.d/目录下

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-5-apparmor%E8%AE%BF%E9%97%AE%E6%8E%A7%E5%88%B6/20220605190438.png)

仔细看这个配置文件和上面手动编写的区别。多了一个flags=(complain)，这个是设置模式的。

# apparmor模式

## complain模式

这个模式不会真的对程序的访问控制进行限制，只是会默默记录下违反该配置文件的操作。

只要在程序执行路径后，大括号前 添加 flags=(complain) 即可

## enforcement

这个模式才真的对程序进行限制，严格执行大括号内的要求。

只要在程序执行路径后，大括号前 没有 flags=(complain) 就是 enforcement模式。

#  apparmor控制细节

对于要手写访问控制文件的童鞋。

## 1.文件系统的访问控制

| 字符  | 含义  |
|---|---|
|r   | - read|
|w   | - write -- conflicts with append|
|a   | - append -- conflicts with write|
|ux  | - unconfined execute|
|Ux  | - unconfined execute -- scrub the environment |
|px  | - discrete profile execute|
|Px   |- discrete profile execute -- scrub the environment|
|cx   |- transition to subprofile on execute|
|Cx   |- transition to subprofile on execute -- scrub the environment|
|ix   |- inherit execute|
|m    |- allow PROT_EXEC with mmap(2) calls|
|l    |- link|
|k    |- lock|

## 2.资源限制

Apparmor可以提供类似系统调用setrlimit一样的方式来限制程序可以使用的资源。要限制资源，可在配置文件中这样写

    set rlimit [resource] <= [value]

其resource代表某一种资源，value代表某一个值，

要对程序可以使用的虚拟内存做限制时，可以这样写：

    set rlimit as<=1M, （可以使用的虚拟内存最大为1M）

注意：Apparmor可以对程序要使用多种资源进行限制（fsize,data,stack,core,rss,as,memlock,msgqueue等），但暂不支持对程序可以使用CPU时间进行限制。

可以参照setrlimit（2）手册页。

## 3.网络控制

Apparmor可以程序是否可以访问网络进行限制，在配置文件里的语法是：

    network [ [domain] [type] [protocol] ]

要允许程序使用在IPv4下使用TCP协议，可以这样写：

    network inet tcp

## 完整例子

    # Last Modified: Wed Feb 12 19:25:31 2020
    #include <tunables/global>

    /path/to/your/binary {
    #include <abstractions/base> //允许进行系统调用，比如exec
    #include <abstractions/apache2-common>//允许进行监听端口等待内容

    //文件系统访问控制
    /lib/x86_64-linux-gnu/ld-*.so mr,
    /usr/bin/vim.tiny mr,
    /var/tmp/** rw,
    /var/tmp/test1.txt r,
    /var/tmp/test2.txt ar,
    /var/tmp/test3.txt wr,

    //资源限制
    set rlimit data <= 100M,
    set rlimit nproc <= 10,
    set rlimit nice <= 5,

    //网络控制
    network,               #allow access to all networking

    network tcp,           #allow access to tcp

    network inet tcp,      #allow access to tcp only for inet4 addresses

    network inet6 tcp,     #allow access to tcp only for inet6 addresses

    network netlink raw,   #allow access to AF_NETLINK SOCK_RAW

    }


# reference

[https://bbs.pediy.com/thread-268215.htm](https://bbs.pediy.com/thread-268215.htm)

[https://blog.csdn.net/culintai3473/article/details/108788329](https://blog.csdn.net/culintai3473/article/details/108788329)

[http://t.zoukankan.com/zlhff-p-5464862.html](http://t.zoukankan.com/zlhff-p-5464862.html)

[https://blog.csdn.net/cooperdoctor/article/details/84062206](https://blog.csdn.net/cooperdoctor/article/details/84062206)

[https://blog.csdn.net/jingemperor/article/details/120061509](https://blog.csdn.net/jingemperor/article/details/120061509)

[https://blog.csdn.net/WUWEIWUYU/article/details/104282647](https://blog.csdn.net/WUWEIWUYU/article/details/104282647)

[https://blog.csdn.net/WUWEIWUYU/article/details/104250687](https://blog.csdn.net/WUWEIWUYU/article/details/104250687)