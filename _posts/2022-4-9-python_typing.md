---
layout: post
tags: [knowledge]
title: "python type hints"
date: 2022-4-9
author: wsxk
comments: true
---

- [写在前面](#写在前面)
- [type hints](#type-hints)


## 写在前面<br> 
不知道大家有没有遇到这样一个情况，在类型A中，使用到了类型B的变量c，然而，在实现类型A时，IDE无法识别到c就是类型B，从而失去了类型补全，导致我们编写代码时出现了困难。<br>

## type hints<br>
类型补全是一个强大的功能，它可以帮助ide识别变量类型，从而方便你进行补全<br>

```python
from typing import Optional

class B:
    def method_in_b(self):
        print("Hello from class B")

class A:
    def __init__(self, b_instance: Optional[B] = None):
        self.b_instance = b_instance

    def use_b(self):
        if self.b_instance:
            self.b_instance.method_in_b()
        else:
            print("b_instance is not set")

# 使用示例
b = B()
a = A(b_instance=b)
a.use_b()
```