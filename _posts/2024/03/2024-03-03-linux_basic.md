---
layout: post
tags : [linux]
date: 2024-3-3
title: "linux I: basic"
author: wsxk
comments: true
---

- [写在前面](#写在前面)
- [1. Command Line](#1-command-line)
- [2. process、program、filesystem、directories](#2-processprogramfilesystemdirectories)
- [3. absolute/relative path](#3-absoluterelative-path)
- [4. environment varibles](#4-environment-varibles)
- [5. symbolic/hard links](#5-symbolichard-links)
- [6. pipes](#6-pipes)
- [7. input/ouput redircetion](#7-inputouput-redircetion)
- [8. 权限管理](#8-权限管理)
- [9. linux常见命令](#9-linux常见命令)
  - [9.1 linux命令使用说明书](#91-linux命令使用说明书)
  - [9.2 常见命令列表](#92-常见命令列表)
  - [9.3 Ctrl+c、Ctrl+z、Ctrl+d](#93-ctrlcctrlzctrld)
- [10. elf文件格式](#10-elf文件格式)


## 写在前面<br>
虽然学习网络安全四年有余了，至今让你提起linux的一些关键概念，还是有些不了解linux系统中的某些关键概念，倍感羞耻<br>

## 1. Command Line<br>
俗称`shell`，是一种与用户交互的界面,本质上也是一个进程（运行中的程序）<br>
用户可以在如下界面输入程序，如`cat flag（寻找叫做cat的程序，以flag作为参数运行）`<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303202219.png)

## 2. process、program、filesystem、directories<br>
**process（进程）指运行中的program（程序）**<br>
**program指的是存储在文件系统(filesystem)中的文件**<br>
`linux`中的文件系统布局通常如下图所示<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303202451.png)
**刚刚说program存储在文件系统中，更准确的来说，存储在文件系统的directories(目录)中**<br>

## 3. absolute/relative path<br>
**绝对路径absolute path**指的是文件在系统中的存放位置<br>
**相对路径relative path**指的是文件相对于当前工作目录下的存放位置<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303202736.png)
关于目录，`.`表示当前目录，`..`表示上一级目录，`/`开头的表示根目录（即最初始的目录）<br>
我们可以通过`ls -l`查看文件的类型，常见类型如下：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303203344.png)

## 4. environment varibles<br>
`environment varibles`是一组键值对（key-value）的集合，在每个程序被执行时，会传递给程序<br>
可以通过`env`程序查看当前的环境变量<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303203058.png)
***但凡我们执行诸如 cat flag这样的命令，然而cat又不在当前目录下，我们也没有输入cat的路径，cat的路径都已经保留在环境变量当中**<br>

## 5. symbolic/hard links<br>
`symbolic links`软链接，指的是`创建一个特殊类型的文件，指向另一个文件`,通过读取这个软链接相当于读取目标文件<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303210124.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303210208.png)
当然，也可以链接`目录`<br>
**注意陷阱：**<br>
**软连接创建时，如果路径使用相对路径时，在使用该软连接进行文件操作时，寻找真的文件时，使用的相对路径会是你当前的动作目录**，所以如果移动了软连接的位置，就用不了了<br>
如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303211454.png)


`hard links`硬链接，与软链接有所不同，创建的是一个指向目标文件的**真实的引用**<br>
如下图所示：<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303211958.png)
从图示中也可以看出，文件类型`-`，表示普通文件<br>
实际原理跟`inode`相关<br>


## 6. pipes<br>
`pipes（管道）`是一种**单向的信息流通机制**<br>
管道分为两种:<br>
```
1. Unnamed pipes（匿名管道）：
最常用于把一个命令的数据 传递到 另一个命令
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303212603.png)
```
2. Named pipes（命名管道）：也被称为 FIFOS
可以使用"mkfifo" 命令来创建
被使用在特定场景下的数据流程
```

## 7. input/ouput redircetion<br>
`输入/输出重定向`指的是 命令的输出不一定就在界面上，可以重定向到文件中， 命令的输入也是，不一定是直接在界面输入，也可以重定向到文件中<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303213020.png)

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-12-30/20240303213100.png)

## 8. 权限管理<br>
详情可见[https://wsxk.github.io/linux%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86/](https://wsxk.github.io/linux%E6%9D%83%E9%99%90%E7%AE%A1%E7%90%86/)<br>

## 9. linux常见命令<br>
### 9.1 linux命令使用说明书<br>
linux中有很多命令可供选择，为了了解命令的使用方式，了解命令说明书的构造就尤为重要<br>
以某个例子为例<br>
```shell
Usage: cpio [OPTION...] [destination-directory]
有 []攘括的 OPTION 表示，该参数是可选参数（可加可不加）
有 ...跟随，表示OPTION可以有一个或多个

除此之外还有其他常见符号
<> 攘括的参数表明该参数不可缺失，比如 command <filename>
{} 攘括的参数表明必须使用其中一个参数， 比如 command {a | b}，表示a、b二选一，
```

### 9.2 常见命令列表<br>
**意想不到的获取文件内容的方法**<br>
```
more ： 浏览文件内容
less ： 浏览文件内容
tail ： 查看文件末尾几行
head ： 查看文件头几行
sort ： 文件行排序后输出
rev ： 倒序输出

vim ： 文本编辑器
emacs ： 文本编辑器
nano ： 文本编辑器

od ： Octal Dump
hd ： Hex Dump
xxd ： 主要用于创建十六进制或二进制的转储

base32: 内容以base32加密后输出
base64: 内容以base64加密后输出

split： 将一个文件的内容 分成若干文件

gzip： 压缩/解压缩 文件 后缀： .gz
bzip2： 压缩/解压缩 文件 后缀： .bz2
zip: 压缩/解压缩文件    后缀： .zip root权限zip压缩后的文件，普通用户也有读权限，可以通过unzip解压
tar： 压缩/解压缩文件 tar -xf flag.tar -O 可以把解压的内容输出到命令行中
ar： 创建、修改静态库 ar -crv flag.a flag 后 使用 ar -p flag.a flag 即可打印flag内容
cpio： 压缩/解压缩 cpio -o <~/1.txt用于创建一个压缩文件
genisoimage： 创建ISO镜像文件 genisoimage -sort /flag

env： 展示linux的环境变量，可以利用env命令的权限执行其他命令  env cat /flag
find：可以通过find找到某文件后，执行某命令 find /flag -exec cat {} \;  或者 find /flag -exec sh -p \; 直接拿root
make： 编译命令 make --eval='target: ; sh -p' target
sh: 正常的sh命令，在执行时，会把euid设置为ruid，如果是sh -p选项的话，则会保留euid的实际值（+s的程序可以用它）
nice： 更改程序优先级 nice cat /flag
timeout：命令程序执行多长时间 timeout 1.0s cat /flag
stdbuf： 重新设置stdin/stdout/stderr的缓冲方式(行缓冲、全缓冲、无缓冲)  stdbuf -iL cat /flag
setarch： 在Linux 中用于设置程序运行时的体系结构相关行为，它可以用于改变当前进程的体系结构报告（如 uname 输出）或设置某些体系结构特定的行为，如ASLR（地址空间布局随机化）  setarch i386 cat /flag
watch： 监控程序执行 watch -x cat /flag （-x和该选项的区别是是否是直接使用watch运行exec来执行命令）
socat： 用于在两个字节流之间建立和传输数据 socat exec:'cat /flag' stdout

whiptail： 图形化界面 whiptail --textbox /flag 20 20
awk： 强大的文本处理工具， awk  '{print $0}' /flag
sed： 强大的文本修改工具， sed -e p -n /flag
grep： 强大的文本过滤工具， 
ed： 行文本编辑器， ed -G /flag，然后输入1p 

usermod: usermod -aG group_ztsmuhtj hacker 将hacker加入group中，加入后需要su hacker刷新会话才会生效
chown： 改变文件属主和属组
chmod： 改变文件的权限(u,g,o)
cat /etc/group: 查看所有用户的所属组
cp： 文件复制 cp /flag /dev/stdout ，将文件内容复制到标准输出
mv： 文件移动 

perl：是perl语言解释器 perl -e 'open(my $fh, "<", "/flag") or die "Cannot open /flag: $!"; print while(<$fh>); close($fh);'
python： python语言解释器
ruby： ruby语言解释器

date： 显示日期，可以 date -f /flag读取文件内容
dmesg： 控制/显示内核环形缓冲区 dmesg -F /flag
wc： 显示文件的单词，行数等等信息  wc --files0-from=/flag
gcc： 编译器 gcc -x c -E /flag
as： 汇编语言编译器 as /flag
wget： 网络获取内容 nc -lp 8888 & wget --post-file=/flag http://127.0.0.1:8888

ssh-keygen: 生成ssh key的命令，可以通过 ssh-keygen -D 来加载任意so
加载so时，可以通过下面的函数
void __attribute__((constructor)) func(){
    printf("test\n");
}
让函数在so加载时(未执行main函数)被执行

tee: 很好用的命令，作用是将输入打印到某个文件和屏幕中，通常和fifo管道配合使用：
mkfifo /tmp/in /tmp/out /tmp/cache
./program1 </tmp/in >/tmp/out  # 运行program1 从in中读取数据，将输出放入out
./program2 </tmp/cache >/tmp/in #运行program2 从cache中读取数据，将输出放入in
cat /tmp/out | tee /tmp/cache # c从out读数据，放入tee命令，tee命令会把数据放入cache，并打印到屏幕
```

### 9.3 Ctrl+c、Ctrl+z、Ctrl+d<br>
```
Ctrl+c：
作用：发送中断信号（SIGINT）给当前正在前台运行的命令，强制其终止执行。

使用场景：当你想要终止当前正在执行的命令，或者当一个命令似乎陷入了死循环而无法响应时
可以使用Ctrl+c来中断命令的执行。
```

```
Ctrl+z：
作用：将当前正在前台运行的命令挂起（Suspend），并放入后台。这意味着该命令的执行将被暂停，但它仍然会保持在内存中。

使用场景：当你想要暂时挂起当前正在执行的命令，并返回到命令行界面进行其他操作时，可以使用Ctrl+z。
比如，你可以使用Ctrl+z暂停一个长时间运行的命令，然后使用fg命令将其恢复到前台继续执行，或者使用bg命令将其放入后台继续执行。
通过jobs查看哪些任务在进行中。
```

```
Ctrl+d：
作用：表示输入流的结束（EOF），通常用于表示用户输入的结束。如果在新的一行中输入Ctrl+d，则表示结束该输入行；如果在命令行中输入Ctrl+d，则表示退出当前的shell会话。

使用场景：当你在命令行中需要输入文件结束符时，可以使用Ctrl+d。在交互式shell中，输入Ctrl+d通常表示退出当前的shell会话，类似于输入exit命令。
```

## 10. elf文件格式<br>
详情见：<br>
[https://wsxk.github.io/elf%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E8%A7%A3%E6%9E%90/](https://wsxk.github.io/elf%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E8%A7%A3%E6%9E%90/)