---
layout: post
tags: [kernel_pwn]
title: "kernel stack 3: canary+smep+kpti绕过"
author: wsxk
date: 2026-9-25
comments: true
---


- [6. canary+smep+kpti: ROP](#6-canarysmepkpti-rop)
  - [6.1 kpti的原理](#61-kpti的原理)
  - [6.2 设置signal handler绕过](#62-设置signal-handler绕过)
  - [6.3 KPTI trampoline + ROP](#63-kpti-trampoline--rop)
- [references](#references)


# 6. canary+smep+kpti: ROP<br>
## 6.1 kpti的原理<br>
kpti诞生是为了阻止meltdown漏洞的，核心原理是维护两套页表：一套跟之前一样，有用户态和内核态全量地址的映射，供内核态使用，**但是将用户态地址空间标记为不可执行**；另一套有用户态全量地址和必要的内核态地址（最小集），供用户态使用。在kpti开启时，在发生系统调用时，会发生页表的切换。<br>
`CR(control register)3寄存器`即页表的物理地址。<br>

启动命令:<br>
```
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,+smep,-smap \
    -kernel bzImage \
    -initrd initramfs.cpio.gz \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -no-reboot \
    -append "console=ttyS0 nokaslr kpti=1 quiet panic=1" \
    -s
```

## 6.2 设置signal handler绕过<br>
这是一个非常有意思的机制。<br>


## 6.3 KPTI trampoline + ROP<br>
绕过方法其实还是ROP，但是因为需要切换页表，我们不知道切换页表要做什么，但是考虑到执行syscall能正常返回，内核里一定有一段关于内核页表切换的代码！<br>



# references<br>
[https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/](https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/)<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>