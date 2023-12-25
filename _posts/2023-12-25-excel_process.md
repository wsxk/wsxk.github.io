---
layout: post
tags: [knowledge]
title: "excel process"
date: 2023-12-25
author: wsxk
comments: true
---

- [1. 前言](#1-前言)
- [2. 代码](#2-代码)
  - [2.1 time\_trans](#21-time_trans)
  - [2.2 excel\_proc](#22-excel_proc)


## 1. 前言<br>
**写excel格式不统一的都是傻Ⅹ**<br>
某位老师让我帮忙处理一下excel表格，真不知道他们怎么改的,格式都tm不统一，死了得了。<br>

## 2. 代码<br>
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