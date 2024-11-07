---
layout: post
tags: [software_build]
title: "linux 下VScode调试 cmake编译程序"
author: wsxk
date: 2023-4-23
comments: true
---

- [写在前面](#写在前面)
- [环境](#环境)
- [.vscode目录文件配置](#vscode目录文件配置)
  - [c\_cpp\_properties.json](#c_cpp_propertiesjson)
  - [launch.json](#launchjson)
  - [tasks.json](#tasksjson)
- [效果展示](#效果展示)


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>

在做某个测试被恶心到了🤢

## 写在前面<br>
VS code调试的时候，默认都是gcc/g++调试，如果你的代码是cmake/make编译的话，如何自动化一键调试+编译呢？<br>

## 环境<br>
ubuntu18.04 （其实其他linux发行版都可以，无所谓<br>
VS code <br>
make<br>
cmake<br>

## .vscode目录文件配置<br>
需要在当前目录下的 `.vscode`目录里添加3个文件<br>
### c_cpp_properties.json<br>
这个文件涉及VScode包含头文件的目录路径:<br>
```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [  //包含目录，这里需要修改，其他默认即可
                "${workspaceFolder}/**",
                "${workspaceFolder}/tests/googletest/googletest/include/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/g++",
            "cStandard": "c11",
            "cppStandard": "gnu++14",
            "intelliSenseMode": "linux-gcc-x64",
            "configurationProvider": "ms-vscode.makefile-tools"
        }
    ],
    "version": 4
}
```
这可以VScode索引到头文件

### launch.json<br>
launch.json是你每次按F5键运行时的默认操作步骤。<br>
```json
// launch.json

{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "C/C++: g++ build active file", 
            "preLaunchTask": "C/C++: g++ build active file",  //在launch之前运行的任务名，即tasks.json中需要运行的命令，这个名字一定要跟tasks.json中的任务名字大小写一致
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/tests/WordFrequencyTest", //需要运行的程序，有时候cmake生成的程序名称和你要调试的文件名不一致
            "args": [],
            "stopAtEntry": false, // 选为true则会在打开控制台后停滞，暂时不执行程序
            "cwd": "${workspaceFolder}", // 当前工作路径：当前文件所在的工作空间
            "environment": [],
            "externalConsole": false,  // 是否使用外部控制台，选false的话
            "MIMode": "gdb",
            "miDebuggerPath": "/usr/bin/gdb",//调试器选择
            "setupCommands": [
                {
                    "description": "Enable pretty-printing for gdb",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }]
}
```

### tasks.json<br>
```json
{
    "version": "2.0.0",
    "options": {
        "cwd": "${workspaceFolder}/tests"
    },
    "tasks": [
        {
            "type": "shell",
            "label": "cmake",
            "command": "cmake",
            "args": [
                "-DCMAKE_BUILD_TYPE=Debug",//很重要，调试按钮，如果没有加，你的cmake页没有设置，就不能下断点了
                ".." //返回到上一级目录运行cmake
            ],
        },
        {
            "label": "make",
            "command": "make",
            "args": [
            ],
            "group": {
                "kind": "build",  //group属性用于定义任务组。kind为build表示这是一个构建任务，isDefault为true表示这是默认的构建任务，即当你执行"Run Build Task"命令（默认快捷键是Ctrl+Shift+B）时运行的任务。
                "isDefault": true
            }
        },
        {
            "label": "C/C++: g++ build active file", //上述的launch.json实际上会调研这个任务
            "dependsOrder": "sequence", //表示要完成这个任务需要一些列的前置任务
            "dependsOn": ["cmake","make"],//前置任务列表
        }
    ]
}
```

## 效果展示<br>
首先是编译的输出<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-4-27-vscode_cmake/20230427133840.png)
随后程序就会停在你下断点的位置。
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2023-2-18-reverse/20230427133736.png)