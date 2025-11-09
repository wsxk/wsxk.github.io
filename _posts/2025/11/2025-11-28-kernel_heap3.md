---
layout: post
tags: [kernel_pwn]
title: "kernel heap 2: msg_msg和pipe_buffer"
author: wsxk
date: 2025-11-28
comments: true
---

- [1. msg\_msg](#1-msg_msg)
- [2. pipe\_buffer](#2-pipe_buffer)




# 1. msg_msg<br>
`msg_msg`是linux提供的一种进程间通信（IPC）的结构体。<br>

```c
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);

ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
```

# 2. pipe_buffer<br>




<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>