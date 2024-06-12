---
layout: post
tags: [web]
title: "Building a Web Server"
author: wsxk
date: 2024-6-8
comments: true
---

- [前言](#前言)
- [1. 构建web server所需的system call与结构体](#1-构建web-server所需的system-call与结构体)
  - [1.1 用汇编来实现上述步骤](#11-用汇编来实现上述步骤)

## 前言<br>
要想构建一个简单的`web server`，最简单的办法当然是使用`python`辣，但是为了能够更清楚的了解`web server`的运行原理，在`linux`上用`assembly`是最合适的~<br>
首先需要了解的一点是：**web server是构建在linux操作系统上的应用程序，其与外界进行互动时，需要让linux操作系统来充当中介**<br>

## 1. 构建web server所需的system call与结构体<br>
```c
int socket(
    int domain,     //socket() creates an endpoint for communication  
    int type,       //and returns a file descriptor
    int protocol    //that refers to that endpoint.
)
```

```c
int bind(
    int sockfd,     //When a socket(2) is created with socket,
    struct sockaddr *addr, //it exists in a name space (address family) but has no address assigned to it. 
    socklen_t addrlen//bind() assigns the address specified by addr to the socket referred to by the file descriptor sockfd.
)
```
这里出现了关键的结构体：<br>
```c
struct sockaddr {
  uint16_t  sa_family;
  uint8_t   sa_data[14];
};
// sockaddr结构体用来描述一个网络连接还是太粗糙了，于是有了下面的改进版本
struct sockaddr_in {
  uint16_t  sin_family;
  uint16_t  sin_port;
  uint32_t  sin_addr;
  uint8_t   __pad[8];
}
// 可以看到sockaddr_in和sockaddr本质上是一个东西，只不过sockaddr_in划分结构体成员更加精细
```

```c
int listen(
    int sockfd, //listen() marks the socket referred to by sockfd as a passive socket,
    int backlog//that is, as a socket that will be used to accept incoming connection requests using accept(2).
)
```

```c
int accept(
    int sockfd, //The accept() system call is used with connection-based socket types (SOCK_STREAM, SOCK_SEQPACKET). 
    struct sockaddr *addr,//It extracts the first connection request on the queue of pending connections for the listening socket, sockfd, 
    socklen_t *addrlen//creates a new connected socket, and returns a new file descriptor referring to that socket.
)
```

一般服务器接收请求的步骤如下:<br>
```c
socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 3
bind(3, 
     {sa_family=AF_INET, 
      sin_port=htons(80),
      sin_addr=inet_addr("0.0.0.0")},
     16)                                 = 0
listen(3, 0)                             = 0
accept(3, NULL, NULL)                    = 4
read(4, 
     "GET /flag HTTP/1.0\r\n\r\n",
     256)                                = 19
open("/flag", O_RDONLY)                  = 5
read(5, "FLAG", 256)                     = 4
write(4, 
      "HTTP/1.0 200 OK\r\n\r\nFLAG",
      27)                                = 27
close(4)  
```


```c
// 上述只处理一条链接，实际上的网络会发送多条链接，解决办法：
socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 3
bind(3, 
     {sa_family=AF_INET, 
      sin_port=htons(80),
      sin_addr=inet_addr("0.0.0.0")},
     16)                                 = 0
listen(3, 0)                             = 0
accept(3, NULL, NULL)                    = 4
fork()                                   = 43            fork()                                   = 0

close(4)                                 = 0
                                                         close(3)                                 = 0
accept(3, NULL, NULL)                    = 4
```

### 1.1 用汇编来实现上述步骤<br>
使用`as -o server.o server.s && ld -o server server.o`命令来完成编译。<br>

```c
# assembler grammar, GNU Assembler(GAS)
.intel_syntax noprefix
.globl _start

.section .data
    acceptfd:
        .long 0
    sockfd:
        .long 0 # safed fd
    sockaddr_in:
        .word 2 # sa_family= AF_INET
        .word 0x5000 # sin_port (htons(bind_poer))
        .long 0x00000000 # sin_addr (inet_addr(bind_address))
        .long 0, 0 # sin_zero
    http_response:
        .asciz "HTTP/1.0 200 OK\r\n\r\n"

.section .text
_start:
    mov rdi, 2      # domain = AF_INET
    mov rsi, 1      # type = SOCK_STREAM (tcp)
    mov rdx, 0      # protocol = 0 (default)
    mov rax, 41     # SYS_socket
    syscall
    mov [sockfd], eax


    xor rdi, rdi
    mov edi, [sockfd]
    lea rsi, [sockaddr_in]
    mov rdx, 16
    mov rax, 49     # SYS_bind
    syscall

    xor rdi, rdi
    mov edi, [sockfd]
    mov rsi, 0
    mov rax, 50     # SYS_listen
    syscall

    xor rdi, rdi
    mov edi, [sockfd]
    mov rsi, 0
    mov rdx, 0
    mov rax, 43     # SYS_accept
    syscall
    mov [acceptfd], eax

    xor rdi, rdi
    mov edi, [acceptfd]
    sub rsp, 0x100
    mov rsi, rsp
    mov rdx, 0x100
    mov rax, 0      # SYS_read
    syscall

    xor rdi, rdi
    mov edi, [acceptfd]
    lea rsi, [http_response]
    mov rdx, 19
    mov rax, 1      # SYS_write
    syscall

    xor rdi, rdi
    mov edi, [acceptfd]
    mov rax, 3      # SYS_close
    syscall

    mov rax, 60     # SYS_exit
    mov rdi, 0
    syscall
```


