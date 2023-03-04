---
layout: post
tags: [iot]
title: "retrowrite 源码分析"
author: wsxk
date: 2023-3-2
comments: true
---

开始分析retrowrite的源码<br>
它的源码因为支持x64和aarch64位的，看起来会比较抽象，因此我把它主要运行代码关于x64的源码扣了出来，方便观看<br>

- [main function](#main-function)


## main function<br>
```python
import argparse
import json
import tempfile
import subprocess
import os
import sys
import traceback
import importlib
from elftools.elf.elffile import ELFFile
from librw_arm64.util.logging import info

from librw_x64.rw import Rewriter
from librw_x64.analysis.register import RegisterAnalysis
from librw_x64.analysis.stackframe import StackFrameAnalysis
from rwtools_x64.asan.instrument import Instrument
from librw_x64.loader import Loader
from librw_x64.analysis import register

bin = "./heap"
outfile = "heap.s"
arch = "x64"
cache = 1
args = ""

Rewriter.detailed_disasm = True

def load_analysis_cache(loader, outfile):
    with open(outfile + ".analysis_cache") as fd:
        analysis = json.load(fd)
    print("[*] Loading analysis cache")
    for func, info in analysis.items():
        for key, finfo in info.items():
            loader.container.functions[int(func)].analysis[key] = dict()
            for k, v in finfo.items():
                try:
                    addr = int(k)
                except ValueError:
                    addr = k
                loader.container.functions[int(func)].analysis[key][addr] = v

def save_analysis_cache(loader, outfile):
    analysis = dict()

    for addr, func in loader.container.functions.items():
        analysis[addr] = dict()
        analysis[addr]["free_registers"] = dict()
        for k, info in func.analysis["free_registers"].items():
            analysis[addr]["free_registers"][k] = list(info)

    with open(outfile + ".analysis_cache", "w") as fd:
        json.dump(analysis, fd)

def analyze_registers(loader):
    StackFrameAnalysis.analyze(loader.container)
    if cache:
        try:
            load_analysis_cache(loader, outfile)
        except IOError:
            RegisterAnalysis.analyze(loader.container)
            save_analysis_cache(loader, outfile)
    else:
        RegisterAnalysis.analyze(loader.container)

def asan(rw, loader):
    analyze_registers(loader)

    instrumenter = Instrument(rw)
    instrumenter.do_instrument()
    instrumenter.dump_stats()


# load binary
loader = Loader("./heap")

if arch == "x64" and loader.is_pie() == False :
    print("***** The x64 version of RetroWrite requires a position-independent executable. *****")
    print("It looks like %s is not position independent" % bin)
    print("If you really want to continue, because you think retrowrite has made a mistake, pass --ignore-no-pie.")
    sys.exit(1)
if arch == "x64" and loader.is_stripped() == True:
    print("The x64 version of RetroWrite requires a non-stripped executable.")
    print("It looks like %s is stripped" % bin)
    print("If you really want to continue, because you think retrowrite has made a mistake, pass --ignore-stripped.")
    sys.exit(1)

# get sections info
slist = loader.slist_from_symtab() #get all section baseinfo
if not ".gcc_except_table" in slist:
    # if there are no exceptions, we never emulate calls
    Rewriter.emulate_calls = False

loader.identify_imports() 
flist = loader.flist_from_symtab() # before loading data sections: get systable baseinfo
loader.load_functions(flist)
loader.load_data_sections(slist, lambda x: x in Rewriter.DATASECTIONS)

reloc_list = loader.reloc_list_from_symtab()
loader.load_relocations(reloc_list)

global_list = loader.global_data_list_from_symtab()
loader.load_globals_from_glist(global_list)

loader.container.attach_loader(loader)

# symbolize
rw = Rewriter(loader.container, "heap.s")
rw.symbolize()

asan(rw, loader) # address sanitizer
rw.dump()
```

单看主代码还是比较简洁的<br>
实际上也符合论文描述。<br>
首先加载二进制程序，并做一些处理，然后符号化，最后插桩。<br>
至于为什么缺少了重新汇编的功能，因为它重新汇编需要依赖本地的编译环境<br>
之后会着重看一下其中几个类型的实现，然后继续补充<br>