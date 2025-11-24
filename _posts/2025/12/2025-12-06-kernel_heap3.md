---
layout: post
tags: [kernel_pwn]
title: "kernel heap 2: msg_msg和pipe_buffer"
author: wsxk
date: 2025-12-06
comments: true
---

- [1. msg\_msg](#1-msg_msg)
  - [1.1 msg常见用法](#11-msg常见用法)
  - [1.2 为什么要介绍msg？](#12-为什么要介绍msg)
- [2. pipe\_buffer](#2-pipe_buffer)
  - [2.1 pipe\_buffer 常见用法](#21-pipe_buffer-常见用法)
  - [2.2 为什么要介绍pipe\_buffer?](#22-为什么要介绍pipe_buffer)




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

## 2.2 为什么要介绍pipe_buffer?<br>
pipe_buffer的定义如下:<br>
```c
struct pipe_buffer {
	struct page *page;
	unsigned int offset, len;
	const struct pipe_buf_operations *ops;//包含函数指针，越界写即可完成控制流劫持！
	unsigned int flags;
	unsigned long private;
};
```

希望人没事~<br>
具体细节会在遇到实际的堆题目后开展~<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>