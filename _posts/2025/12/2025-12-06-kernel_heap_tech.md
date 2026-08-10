---
layout: post
tags: [kernel_pwn]
title: "kernel heap 利用技巧: msg_msg和pipe_buffer"
author: wsxk
date: 2025-12-06
comments: true
---

todo: `mas_msg`和`pipe_buffer`在ctf/linux kernel漏洞利用的实际应用。<br>

- [1. msg\_msg](#1-msg_msg)
  - [1.1 msg常见用法](#11-msg常见用法)
  - [1.2 为什么要介绍msg？](#12-为什么要介绍msg)
- [2. pipe\_buffer](#2-pipe_buffer)
  - [2.1 pipe\_buffer 常见用法](#21-pipe_buffer-常见用法)
  - [2.2 为什么要介绍pipe\_buffer?](#22-为什么要介绍pipe_buffer)
- [3. 具体案例](#3-具体案例)


之前提到，在正常的linux kernel环境下，堆风水布局是十分困难的。主要原因是内核大都使用`kmalloc`来申请内存，有漏洞的驱动大概率也是用`kmalloc`申请的内存。<br>
但是`msg_msg`和`pipe_buffer`给我们提供了一个有效的进行堆布局的方法.<br>

# 1. msg_msg<br>
`msg_msg`是linux提供的一种进程间通信（IPC）的结构体。<br>


## 1.1 msg常见用法<br>
msg在linux本质上是通过内核消息队列来进行通信的。<br>
主要函数如下:<br>
```c
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);

ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
```

可以看一下demo：<br>
```c
//msg_send.c 用于发送msg
#include <sys/types.h>
#include <sys/ipc.h>
#include <stdlib.h>
#include <string.h>
#include <sys/msg.h>
#include <stdio.h>

int main(){
    // prepare message
    size_t target_cache_size = 128;
    struct msgbuf{
        long mtype;
        char message[100];
    };
    size_t msg_sz = target_cache_size-0x30;
    struct msgbuf msgbuf;
    strcpy(msgbuf.message,"hello world!");
    
    puts("before creating messages");
    getchar();

    int msqid;
    int key = ftok(".",0);
    msqid = msgget(key,0666| IPC_CREAT);
    for(int i=0;i<20;i++){
        msgbuf.mtype = i+1;//注意mtype不能为0，0会出问题。
        msgsnd(msqid,&msgbuf,msg_sz,0);
    } 
    puts("after creating messages");
    getchar();
}
```

```c
//msg_recv.c 用于接收msg
#include <sys/types.h>
#include <sys/ipc.h>
#include <stdlib.h>
#include <string.h>
#include <sys/msg.h>
#include <stdio.h>

int main(){
    // prepare message
    size_t target_cache_size = 128;
    struct msgbuf{
        long mtype;
        char message[100];
    };
    size_t msg_sz = target_cache_size-0x30;
    struct msgbuf msgbuf;
    strcpy(msgbuf.message,"hello world!");
    
    puts("before creating messages");
    getchar();

    int msqid;
    int key = ftok(".",0);
    msqid = msgget(key,0666);
    for(int i=0;i<20;i++){
        msgrcv(msqid,&msgbuf,msg_sz,0,0);
        printf("received with type: %d, content: %s\n",msgbuf.mtype,msgbuf.message);
    } 
    puts("after creating messages");
    getchar();
}
```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251111220633.png)

## 1.2 为什么要介绍msg？<br>
还记得上一节[https://wsxk.github.io/kernel_heap2/](https://wsxk.github.io/kernel_heap2/)中提到的理想的堆布局构造对象吗？<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20251111220739.png)
首先看内核中`msg_msg`对象的定义:<br>
```c
/* one msg_msg structure for each message */
// 每个消息的头，最多可容纳0x1000-0x30 = 0xfd0个用户数据
struct msg_msg {
	struct list_head m_list;
	long m_type;
	size_t m_ts;		/* message text size */
	struct msg_msgseg *next;   //****完美满足第3点！ptr可覆盖！****
	void *security;
	/* the actual message follows immediately */
};

//m_list contains pointers to messages in the message queue 
//m_ts determines this size of the message text

// 消息段，最多可容纳0x1000-0x08 = 0xff8个用户数据，利用时，因为msg_msg的字段太多，因此大多数利用都是依靠msg_msgseg来实现
struct msg_msgseg {
    struct msg_msgseg *next;   // 0x00
    char data[];               // 0x08
};
```



我们接下来看一下msg_msg是如何分配的：<br>
```c
static struct msg_msg *alloc_msg(size_t len)
{
	struct msg_msg *msg;
	struct msg_msgseg **pseg;
	size_t alen;

	alen = min(len, DATALEN_MSG);
	msg = kmalloc(sizeof(*msg) + alen, GFP_KERNEL_ACCOUNT);//0x28+8（mtype）+实际消息长度
  //****完美满足第一点！size可控！！！****
}
```

**在调用msgrcv函数时，发生如下事件：**<br>
```c
struct msg_msg *copy_msg(struct msg_msg *src, struct msg_msg *dst) {
	struct msg_msgseg *dst_pseg, *src_pseg;
	size_t len = src->m_ts;
	size_t alen;
	if (src->m_ts > dst->m_ts)
		return ERR_PTR(-EINVAL);
	alen = min(len, DATALEN_MSG);
	memcpy(dst + 1, src + 1, alen);//****完美满足第二点！绕过harden_usercopy!!!****

}
//copy_msg can be triggered via  msgrcv(msgqid, msgp, msgsz, 0, MSG_COPY);
```

# 2. pipe_buffer<br>
## 2.1 pipe_buffer 常见用法<br>
`pipe_buffer`也是内核IPC通信的方法之一，通过管道`pipe`来进行通信；<br>
在用户态创建`pipe`后，内核就会维护`pipe_buffer`<br>
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    int pipe_fd[2];
    const char message[] = "hello from pipe_buffer";
    char buffer[128] = {0};
    size_t message_len = strlen(message);
    ssize_t written;
    ssize_t received;

    /* pipe_fd[0] is the read end; pipe_fd[1] is the write end. */
    if (pipe(pipe_fd) == -1) {
        perror("pipe");
        return EXIT_FAILURE;
    }

    written = write(pipe_fd[1], message, message_len);
    if (written == -1) {
        perror("write");
        close(pipe_fd[0]);
        close(pipe_fd[1]);
        return EXIT_FAILURE;
    }

    /* No more data will be written. A later read can therefore see EOF. */
    close(pipe_fd[1]);

    received = read(pipe_fd[0], buffer, sizeof(buffer) - 1);
    if (received == -1) {
        perror("read");
        close(pipe_fd[0]);
        return EXIT_FAILURE;
    }

    buffer[received] = '\0';
    printf("write: %zd bytes\n", written);
    printf("read:  %zd bytes\n", received);
    printf("data:  %s\n", buffer);

    close(pipe_fd[0]);
    return EXIT_SUCCESS;
}

```
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2026-4-26/20260810212403.png)

## 2.2 为什么要介绍pipe_buffer?<br>
pipe_buffer的定义如下:<br>
```c
struct pipe_buffer {
    /* 0x00 */ struct page *page;
    /* 0x08 */ unsigned int offset;
    /* 0x0c */ unsigned int len;
    /* 0x10 */ const struct pipe_buf_operations *ops;
    /* 0x18 */ unsigned int flags;
    /* 0x1c */ unsigned int padding;
    /* 0x20 */ unsigned long private;
}; /* sizeof = 0x28 */
```

# 3. 具体案例<br>
todo，等到复现linux cve的时候应该就用上了<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>