---
layout: post
tags: [re]
title: "pyelftools 解析elf文件符号"
date: 2023-3-25
author: wsxk
comments: true
---

- [前言](#前言)
- [前提条件](#前提条件)
- [dwarf](#dwarf)
  - [DIE](#die)
- [利用pyelftools编写code](#利用pyelftools编写code)


## 前言<br>
通常情况下，我们通过二进制文件`symbol table`获取符号信息，从而可以得知函数名称以及函数地址等等信息，但是，我们不能更细粒度得获取例如函数参数等等细节。<br>
解决方案是，可以利用dwarf格式存储在elf文件某个特定section的调试信息来获取更详细的信息。<br>

## 前提条件<br>
> 1. elf文件
> 2. 带有调试信息(-g)
> 3. 未经过strip操作去符号

## dwarf<br>
DWARF 是一种用于表示调试信息的标准文件格式，主要应用于编程语言和编译器中。DWARF 的主要目的是在编译过程中生成调试信息，以便在执行时为调试器提供源代码级别的详细信息。这包括变量名、类型、函数原型、行号、地址等。<br>
当程序员编译一个程序时，编译器（如 GCC 或 Clang）可以生成带有 DWARF 调试信息的二进制文件。这些调试信息通常存储在目标文件（如 ELF 文件）的特定节中。当使用调试器（如 GDB 或 LLDB）调试程序时，调试器可以解析这些 DWARF 信息，将程序执行状态映射回源代码，从而帮助程序员更好地理解程序在执行过程中发生了什么。<br>
DWARF 的一些主要特点包括：<br>

    1. 平台无关性：DWARF 是一个通用的调试信息格式，可以用于多种处理器架构、操作系统和目标文件格式。
    
    2. 可扩展性：DWARF 具有很高的可扩展性，可以支持不同的编程语言特性、编译器优化和调试器功能。

    3. 高效性：DWARF 使用紧凑的二进制编码，以节省存储空间和提高解析速度。

DWARF 已经成为现代编译器和调试器的事实标准，广泛应用于 Linux、macOS 等操作系统的软件开发和调试。<br>
### DIE<br>
DIE（Debugging Information Entry）是 DWARF 调试信息中的基本构建块。DIE 描述了程序中的一个实体，例如一个变量、类型或函数。DIE 由一系列属性（attributes）组成，这些属性提供了有关实体的详细信息，如名称、类型、地址、行号等。<br>
DIE 在 DWARF 调试信息中以树状结构组织。根据实体之间的关系，DIE 可以具有父节点、子节点和兄弟节点。例如，一个函数的 DIE 可能包含子节点，分别表示函数的参数、局部变量和返回类型。这种树状结构使得调试器能够更好地理解程序的结构和组织。<br>
每个 DIE 都有一个唯一的标签（tag），表示实体的类型。例如：<br>
> 1. DW_TAG_variable：表示一个变量
> 2. DW_TAG_base_type：表示一个基本类型，如 int、float 等
> 3. DW_TAG_pointer_type：表示一个指针类型
> 4. DW_TAG_array_type：表示一个数组类型
> 5. DW_TAG_structure_type：表示一个结构体类型
> 6. DW_TAG_union_type：表示一个联合体类型
> 7. DW_TAG_enumeration_type：表示一个枚举类型
> 8. DW_TAG_subprogram：表示一个函数或方法
调试器可以根据 DIE 的标签和属性提取有关程序实体的详细信息，从而实现源代码级别的调试功能。<br>

## 利用pyelftools编写code<br>
```python
from elftools.elf.elffile import ELFFile
from elftools.elf.sections import SymbolTableSection
from elftools.dwarf.descriptions import describe_DWARF_expr
from elftools.dwarf.die import DIE

BINARY_FILE = 'XML_Parser.elf'
binary_function= {}

def get_function_addresses(elffile,fname):
    # 寻找符号表
    symtab = None
    for section in elffile.iter_sections():
        if isinstance(section, SymbolTableSection):
            symtab = section
            break
    # 如果找到符号表，提取函数名称和地址
    if symtab:
        for symbol in symtab.iter_symbols():
            #if symbol.entry.st_info.type == 'STT_FUNC' and fname == symbol.name:
            if fname == symbol.name:
                return symbol.entry.st_value
    else:
        print(f"No symbol table found in binary_file")
        return None

def process_file(filename):
    with open(filename, 'rb') as f:
        elffile = ELFFile(f)
        # 获取 DWARF 调试信息
        dwarf_info = elffile.get_dwarf_info()
        # 遍历编译单元
        for cu in dwarf_info.iter_CUs():
            top_DIE = cu.get_top_DIE()
            # 递归遍历 DIE (Debugging Information Entry)
            process_DIE(top_DIE,elffile)


def process_DIE(die,elffile):
    if die.tag == 'DW_TAG_subprogram':
        # 处理函数原型
        process_function(die,elffile)
    for child in die.iter_children():
        process_DIE(child,elffile)

def process_function(die,elffile):
    if 'DW_AT_name' not in die.attributes:
        return
    # 获取函数名
    name = die.attributes['DW_AT_name'].value.decode('utf-8')
    # 获取参数列表
    params = []
    for child in die.iter_children():
        if child.tag == 'DW_TAG_formal_parameter':
            params.append(process_param(child))
    # 打印函数原型
    address= get_function_addresses(elffile,name)
    true_param =''
    if len(params)>=1:
        true_param = ', '.join(params)
    else:
        true_param = 'no_arguments'
    print(f"{name}({true_param}) address:{address}")
    #binary_function[func_name] = [params,get_function_addresses(elffile,name)]

def process_param(die):
    if 'DW_AT_name' not in die.attributes or 'DW_AT_type' not in die.attributes:
        return ""
    name = die.attributes['DW_AT_name'].value.decode('utf-8')
    type_offset = die.attributes['DW_AT_type'].value
    type_die = get_DIE_from_offset(die.dwarfinfo, type_offset)
    type_name = process_type(type_die)
    return f"{type_name} {name}"

def get_DIE_from_offset(dwarf_info, offset):
    for cu in dwarf_info.iter_CUs():
        for die in cu.iter_DIEs():
            if die.offset == offset:
                return die
    return None


def process_type(die):
    if die is not None and 'DW_AT_name' in die.attributes:
        return die.attributes['DW_AT_name'].value.decode('utf-8')
    return ""

if __name__ == "__main__":
# 替换 'input.elf' 为您的 ELF 文件名
    process_file(BINARY_FILE)
    # for each in binary_function:
    #     print("{each} {binary_function[each]}")
```