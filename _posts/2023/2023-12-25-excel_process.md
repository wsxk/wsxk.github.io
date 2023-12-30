---
layout: post
tags: [knowledge]
title: "excel process"
date: 2023-12-25
author: wsxk
comments: true
---

- [1. 前言](#1-前言)
- [2. 代码 version 1](#2-代码-version-1)
  - [2.1 time\_trans](#21-time_trans)
  - [2.2 excel\_proc](#22-excel_proc)
- [3. 代码 version2](#3-代码-version2)
  - [3.1 time\_trans.py](#31-time_transpy)
  - [3.2 number\_trans.py](#32-number_transpy)
  - [3.3 excel\_proc.py](#33-excel_procpy)


## 1. 前言<br>
***场景：有3个不同的excel文件A,B,C，记录了老师的姓名、生日、电话号码等信息，需要将这些信息整合到一个excel文件中；其中，如果B,C中出现了A中相同的老师，需要以A的信息为准，舍弃B,C的信息；如果B,C中出现了A没有的老师，需要进行整合***<br>
**写excel格式不统一的都是傻Ⅹ**<br>
某位老师让我帮忙处理一下excel表格，真不知道他们怎么改的,格式都tm不统一，死了得了。<br>

## 2. 代码 version 1<br>
### 2.1 time_trans<br>
`time_trans.py主要用作日期转换，因为有的傻叉，日期格式不统一，我服了`<br>
```python
import datetime

def int2str(input):
    # 定义Excel日期序列号
    excel_date = input

    # Excel和Python日期系统之间的差异（Python的1是0001-01-01，而Excel的1是1900-01-01）
    # Excel错误地将1900年当作了闰年，所以需要减去1天
    delta = datetime.timedelta(days=excel_date - 1)

    # Excel的起始日期
    start_date = datetime.datetime(1899, 12, 31)

    # 计算实际日期
    actual_date = start_date + delta

    # 转换成字符串
    date_str = actual_date.strftime('%m月%d日')

    return date_str

def datetime2str(input):
    output = input.strftime('%m月%d日')
    return output

if __name__ == "__main__":
    # 定义Excel日期序列号
    excel_date = 43468

    # Excel和Python日期系统之间的差异（Python的1是0001-01-01，而Excel的1是1900-01-01）
    # Excel错误地将1900年当作了闰年，所以需要减去1天
    delta = datetime.timedelta(days=excel_date - 1)

    # Excel的起始日期
    start_date = datetime.datetime(1899, 12, 31)

    # 计算实际日期
    actual_date = start_date + delta

    # 转换成字符串
    date_str = actual_date.strftime('%m月%d日')

    print(date_str)

    print(type(actual_date)== datetime.datetime)
    print(type( datetime.datetime(2023, 6, 6, 0, 0))==datetime.datetime)
```

### 2.2 excel_proc<br>
`excel_proc.py负责主逻辑`<br>
```python
import openpyxl
import datetime
from time_trans import datetime2str, int2str

count = 0

# 读取excel文件
workbook_main = openpyxl.load_workbook('./excel_process/main.xlsx')
workbook_sub = openpyxl.load_workbook('./excel_process/material1.xlsx')
workbook_sub2 = openpyxl.load_workbook('./excel_process/material2.xlsx')

# 获取sheet
sheet_main = workbook_main['Sheet1']
sheet_sub = workbook_sub['Sheet1']
sheet_sub2 = workbook_sub2['Sheet1']

# 获取main的值
def get_value(sheet, check_value, position):
    # print("check_value:{}".format(check_value))
    if type(check_value) == int:
        check_value = str(check_value)
    for row in sheet.rows:
        cell = row[position].value
        if type(cell) == int:
            cell = str(cell)
        if cell == check_value:
            return False
        else:
            continue
    return True

# 获取所需行的单元值
def get_row(row, position_list):
    global count
    list1 = []
    for i in position_list:
        if type(row[i].value) == datetime.datetime:
            print("type is datetime:{}".format(row[i].value))
            list1.append(datetime2str(row[i].value))
        else:
            if type(row[i].value) == int and row[i].value < 100000:
                print("type is int:{}".format(row[i].value))
                list1.append(int2str(row[i].value))
            else:
                print("type:{} value:{}".format(type(row[i].value),row[i].value))
                list1.append(row[i].value)
    print("append {}".format(list1))
    count += 1
    return list1

# process material1
for row in sheet_sub.rows:
    cell = row[4].value
    if cell == None:
        continue
    print("process material1: phone number:{}".format(cell))
    if  get_value(sheet_main, cell, 2) == False:
        continue
    else:
        sheet_main.append(get_row(row,[3,1,4]))

# process matirial2
for row in sheet_sub2.rows:
    cell = row[3].value
    if cell == None:
        continue
    print("process material2: type:{}, phone number:{}".format(type(cell),cell))
    if get_value(sheet_main, cell, 2)  == False:
        continue
    else:
        sheet_main.append(get_row(row,[1,2,3]))

workbook_main.save('test.xlsx')
print("change success!")
print("add {} rows".format(count))
```

## 3. 代码 version2<br>
上一个版本太过粗糙，耦合严重，不利于后续利用，重新思考了架构后，重写了代码。<br>
### 3.1 time_trans.py<br>
**将未知的时间类型转换成统一的str类型时间格式**<br>
```python
import datetime

def unknown_date2str(input):
    if type(input) == datetime.datetime:
        return datetime2str(input)
    elif type(input) == int and input < 100000:
        return int2str(input)
    else:
        return input

def int2str(input):
    # 定义Excel日期序列号
    excel_date = input

    # Excel和Python日期系统之间的差异（Python的1是0001-01-01，而Excel的1是1900-01-01）
    # Excel错误地将1900年当作了闰年，所以需要减去1天
    delta = datetime.timedelta(days=excel_date - 1)

    # Excel的起始日期
    start_date = datetime.datetime(1899, 12, 31)

    # 计算实际日期
    actual_date = start_date + delta

    # 转换成字符串
    date_str = actual_date.strftime('%m月%d日')

    return date_str

def datetime2str(input):
    output = input.strftime('%m月%d日')
    return output

if __name__ == "__main__":
    # 定义Excel日期序列号
    excel_date = 43468

    # Excel和Python日期系统之间的差异（Python的1是0001-01-01，而Excel的1是1900-01-01）
    # Excel错误地将1900年当作了闰年，所以需要减去1天
    delta = datetime.timedelta(days=excel_date - 1)

    # Excel的起始日期
    start_date = datetime.datetime(1899, 12, 31)

    # 计算实际日期
    actual_date = start_date + delta

    # 转换成字符串
    date_str = actual_date.strftime('%m月%d日')

    print(date_str)

    print(type(actual_date)== datetime.datetime)
    print(type( datetime.datetime(2023, 6, 6, 0, 0))==datetime.datetime)
```

### 3.2 number_trans.py<br>
**将未知的号码类型转换成统一的str类型号码格式**<br>
```python
def unknown_num2str(input):
    if type(input) == int:
        return str(input)
    else:
        return input
```

### 3.3 excel_proc.py<br>
**主逻辑**<br>
```python
import openpyxl
import datetime
from time_trans import unknown_date2str
from number_trans import unknown_num2str

# 读取excel文件
workbook_main = openpyxl.load_workbook('./excel_process/main.xlsx')
workbook_sub = openpyxl.load_workbook('./excel_process/material1.xlsx')
workbook_sub2 = openpyxl.load_workbook('./excel_process/material2.xlsx')

# 获取sheet
sheet_main = workbook_main['Sheet1']
sheet_sub = workbook_sub['Sheet1']
sheet_sub2 = workbook_sub2['Sheet1']

# process phone number
def get_main_value(sheet, check_value, position):
    check_value = unknown_num2str(check_value)
    for row in sheet.rows:
        cell = unknown_num2str(row[position].value)
        if cell == check_value:
            return False
        else:
            continue
    return True

# 获取添加行的单元值
def get_row(row, position_list):
    list1 = []
    for i in position_list:
        if type(row[i].value) == datetime.datetime or (type(row[i].value) == int and row[i].value < 100000):
            list1.append(unknown_date2str(row[i].value))
        else:
            list1.append(unknown_num2str(row[i].value))
    print("append {}".format(list1))
    return list1

# process material1 get_phone_number
for row in sheet_sub.rows:
    cell = row[4].value
    if cell == None:
        continue
    # print("process material1: phone number:{}".format(cell))
    if  get_main_value(sheet_main, cell, 2) == False:
        continue
    else:
        sheet_main.append(get_row(row,[3,1,4]))

# process matirial2
for row in sheet_sub2.rows:
    cell = row[3].value
    if cell == None:
        continue
    # print("process material2: type:{}, phone number:{}".format(type(cell),cell))
    if get_main_value(sheet_main, cell, 2)  == False:
        continue
    else:
        sheet_main.append(get_row(row,[1,2,3]))

workbook_main.save('test1.xlsx')
print("change success!")
```

