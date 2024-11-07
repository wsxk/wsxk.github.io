---
layout: post
tag: [re]
title: "idapython常用api"
author: wsxk
date: 2022-10-1
comments: true
---

之前看其他大佬用脚本自动化修改ida中的数据库，觉得十分牛逼，于是想着把idapython学一下，事实证明，学完后大受震撼，一下子就打开了新大陆。<br>
PS: 所有的函数都基于新版idapython（从ida支持python3开始）<br>
详情可以查官方文档[https://hex-rays.com/products/ida/support/idapython_docs/](https://hex-rays.com/products/ida/support/idapython_docs/)<br>

- [导入库](#导入库)
- [汇编操作](#汇编操作)
- [get\&patch操作](#getpatch操作)
- [段操作](#段操作)
- [函数操作](#函数操作)
- [搜索操作](#搜索操作)
- [交叉引用](#交叉引用)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>


## 导入库<br>
开局先导入<br>
```python
import idc
import idaapi
import idautils
import ida_funcs
import ida_kernwin
import ida_search
import ida_xref 
```

## 汇编操作<br>
ida可以支持写脚本批量处理汇编语句
```python
idc.generate_disasm_line(addr,flags) # 获得addr位置的一条汇编语句
idc.print_operand(addr,index) # 获得addr位置的汇编语句的 第index个操作数 
idc.print_insn_mnem(addr) # 获得addr位置的汇编语句的 操作符
idc.get_operand_type(addr,index) #获得addr位置的汇编语句的第index个操作数的 类型
idc.get_operand_value(addr,index) #获得addr位置的汇编语句的第index操作数的值
idaapi.get_imagebase() #获得程序基地址
idc.here() # 获得当前光标所在的位置
idc.next_head(addr) #addr位置的下一条汇编语句的位置
idc.prev_head(addr) #addr位置的上一条汇编语句的位置
```

## get&patch操作<br>
ida支持提取数据和修改数据<br>
```python
idc.get_db_byte(addr) # 获得从addr开始的一字节
idc.get_wide_word(addr) #获得从addr开始的2字节
idc.get_wide_dword(addr) # 4字节
idc.get_qword(addr) #8字节
idc.get_bytes(addr,size,use_dbg) #从addr获取size个字节后形成的bytes对象
idc.patch_byte(addr,value)#将addr处的一字节改成value
idc.patch_word(addr,value)# 2bytes
idc.patch_dword(addr,value)# 4bytes
idc.patch_qword(addr,value)#8bytes
```
## 段操作<br>
```python
idc.get_segm_name(addr) #获得addr所在segment的名称
idc.get_segm_start(addr) #获得addr所在segment的起始位置
idc.get_segm_end(addr) # 末尾位置
idc.get_first_seg() #获得程序的第一个段的起始地址
idc.get_next_seg(addr)#获得addr所在段的下一个段的起始地址
segments=idautils.Segments()#获得当前程序所有段的迭代器
for each in segments:
    print(each) #打印每个段的起始地址
```

## 函数操作<br>
```python
Functions(startaddr,endaddr)#获得指定地址间的所有函数
idc.get_func_name(addr) #获得addr所在函数的函数名
idc.get_func_cmt(addr,repeatable)#获取函数的注释 repeatable通常为0
idc.set_func_cmt(addr,strings,repeatable) #设置函数注释
idc.choose_func(title) # 弹出窗口，由用户选择一个func
idc.get_func_off_str(addr)#将addr表现为 函数名称+偏移的形式
idc.find_func_end(addr)#找到addr所在函数的结尾
idc.set_func_name(addr,name,0)#设置addr所在函数的新名称
idc.get_prev_func(addr)#上一个函数的起始位置
idc.get_next_func(addr)#下一个函数的起始位置
ida_funcs.set_func_start(addr,newstart)#重新设置函数的起始位置
ida_funcs.set_func_end(addr,newend)#重新设置函数的末尾位置
```

## 搜索操作<br>
```python
idc.find_binary(addr,flags,search_str)#从addr开始，按照flags规定的方式，搜索search_str(比如"e9 d0 a0")所在的位置
idc.find_data(addr,flags)#从addr开始，按照flags的方式，搜索第一个发现的data所在地址
idc.find_code(addr,flags)#~ code所在地址
idc.kernwin.jumpto(addr) #光标跳转到addr处
```

## 交叉引用<br>
```python
CodeRefsTo(addr,bool flow)#查看谁引用了addr，flow通常为True
CodeRefsFrom(addr,bool flow)#addr引用了哪个函数
DataRefsTo(addr) #addr被谁引用
DataRefsFrom(addr)#addr引用了哪个data地址
```