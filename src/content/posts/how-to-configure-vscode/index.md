---
title: Visual Studio Code配置指南
published: 2024-10-19
description: '面向初学者的VS Code配置教程'
image: ''
tags: [教程, Visual Studio Code]
category: '教程'
draft: true 
lang: ''
---

## 前言
本文将手把手教你如何在Windows上配置Visual Studio Code（以下简称VS Code）并将其用于编写、调试运行单文件C/C++程序，如果你的操作系统环境是Linux或Mac OS，请阅读[官方文档](https://code.visualstudio.com/Docs)进行配置。 

## 安装Visual Studio Code
在[官网](https://code.visualstudio.com/)根据你的操作系统下载Visual Studio Code，如果你是Windows 7系统，在[这里](https://code.visualstudio.com/updates/v1_70)下载最后一个支持Windows 7的版本。

## 安装MinGW-W64
事实上，VS Code是一个编辑器而非IDE，只有编辑器是无法像Visual Studio、CLion等IDE一样编译C/C++代码的，因此我们需要安装一个编译器。在Windows上我们推荐安装MinGW-W64（注意是MinGW-W64不是MinGW）作为编译器。MinGW是GNU工具（包括编译器GCC、GNU binutils和调试器GDB等）在Windows上的一个移植，但是它仅支持32位，而MinGW-W64是MinGW的一个分支，它提供了64位支持[^1]。

[^1]: [[科普][FAQ]MinGW vs MinGW-W64及其它](https://github.com/FrankHB/pl-docs/blob/master/zh-CN/mingw-vs-mingw-v64.md)

MinGW-W64有多个发行版本如mingw-builds、MSYS2、LLVM-MinGW等，这里我们选择mingw-builds。在[GitHub](https://github.com/niXman/mingw-builds-binaries/releases)上选择一个版本下载，本文选择**x86_64-14.2.0-release-posix-seh-ucrt-rt_v12-rev0.7z**作为安装版本。

下载完后得到一个拓展名为.7z的压缩包，解压后得到的就是MinGW-W64本体，把它放在一个你能找到的位置比如C盘根目录下面（不管放到哪里，路径不要含有中文防止出现奇怪的错误）。接下来将它添加到Path，右键“此电脑”-属性-高级系统设置-环境变量，选择系统变量下的Path项点击编辑-新建，将刚才解压得到的MinGW文件夹里的bin文件夹路径写进去（如：C:\mingw64\bin）保存后MinGW就被添加到了Path里面。

打开Windows终端（或者你也可以在VS Code里点击终端-新建终端）输入
```
gcc -v
```
回车后你会看见关于MinGW-W64的版本信息，如果提示“'gcc'不是内部或外部命令，也不是可运行的程序或批处理文件。”可尝试重启电脑，若仍出现该消息，说明没有成功添加到Path，请检查是不是出现了拼写错误或路径错误。

## 配置Visual Studio Code

为了便于使用，我们通常会在一个文件夹内编写代码，你可以选择一个没有中文的路径新建一个文件夹用于编写代码，然后在其中新建一个.vscode文件夹。在.vscode文件夹下我们新建tasks.json和launch.json两个文件，在tasks.json中粘贴如下内容：
```json
{
    "tasks": [
        {
            "type": "cppbuild",
            "label": "g++ build and debug", // 编译C++的task，供launch.json使用
            "command": "C:/mingw64/bin/x86_64-w64-mingw32-g++", // 这里写存放mingw的路径
            "args": [ // 下面是编译参数
                "-fdiagnostics-color=always", // 强制编译器在输出错误和警告信息时使用颜色，便于阅读
                "-g", // 生成调试信息
                "-Wall", // 启用所有警告
                "-std=c++20", // 使用C++20作为标准进行编译
                "-lm", // 链接到数学库
                "${file}",
                "-o",
                "${fileDirname}\\${fileBasenameNoExtension}.exe"
            ],
            "options": {
                "cwd": "C:/mingw64/bin"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "presentation": {
                "echo": true,
                "reveal": "always",
                "focus": true,
                "panel": "new"
            },
            "detail": "调试器生成的任务。"
        },
        {
            "type": "cppbuild",
            "label": "gcc build and debug", // 编译C的task，供launch.json使用
            "command": "C:/mingw64/bin/x86_64-w64-mingw32-gcc", // 这里写存放mingw的路径
            "args": [
                "-fdiagnostics-color=always", // 强制编译器在输出错误和警告信息时使用颜色，便于阅读
                "-g", // 生成调试信息
                "-Wall", // 启用所有警告
                "-std=c17", // 使用C17作为标准进行编译
                "-lm", // 链接到数学库
                "${file}",
                "-o",
                "${fileDirname}\\${fileBasenameNoExtension}.exe"
            ],
            "options": {
                "cwd": "C:/mingw64/bin"
            },
            "problemMatcher": [
                "$gcc"
            ],
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "detail": "调试器生成的任务。"
        }
    ],
    "version": "2.0.0"
}
```

在launch.json中粘贴如下内容：
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "g++", // 你可以自定义这个名字
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}/${fileBasenameNoExtension}.exe",
            "args": [], // 要传递给程序的命令行参数，此处为空
            "stopAtEntry": false, // 设置为true会在程序入口处停止
            "cwd": "${fileDirname}",
            "environment": [], // 可以在这里定义环境变量，此处为空
            "externalConsole": false, // 设置为true会使用外部终端来运行程序，而不是VSCode内部终端
            "MIMode": "gdb",
            "miDebuggerPath": "C:\\mingw64\\bin\\gdb.exe", // 这里写存放mingw的路径
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "g++ build and debug" // 在启动调试之前要执行的编译任务，要与tasks.json里的label对应
        },
        {
            "name": "gcc",
            "type": "cppdbg",
            "request": "launch",
            "program": "${fileDirname}/${fileBasenameNoExtension}.exe",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${fileDirname}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "gdb",
            "miDebuggerPath": "C:\\mingw64\\bin\\gdb.exe",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                },
                {
                    "description": "将反汇编风格设置为 Intel",
                    "text": "-gdb-set disassembly-flavor intel",
                    "ignoreFailures": true
                }
            ],
            "preLaunchTask": "gcc build and debug"
        }
    ]
}
```

现在我们可以一键编译、运行和调试C/C++代码了，打开C/C++文件，点击左侧栏的运行和调试按钮，我们可以选择g++或gcc两个配置选项。对于C++我们选择g++，对于C我们选择gcc，点击绿色三角即可开始编译调试，或者你也可以通过键盘快捷键F5进行调试或者Ctrl+F5以非调试模式运行（此时断点不生效）。实际上tasks.json和launch.json有更多的配置选项，具体可以到[官方文档](https://code.visualstudio.com/docs/cpp/launch-json-reference)内查看。

:::warning
请务必选择正确的配置选项运行，否则将出现编译器报错。

如果终端提示
```
无法生成和调试，因为活动文件不是 C 或 C++ 源文件。

 *  终端进程启动失败(退出代码: -1)。 
 *  终端将被任务重用，按任意键关闭。
```
并弹出窗口，说明你尝试编译运行的不是C/C++文件。
:::

## 使用clangd作为语言服务器
LSP即Language Server Protocol，是由微软定义的编辑器或IDE与语言服务器之间使用的协议，该协议提供自动补全、转到定义、查找所有引用等语言功能，同时具有较小的性能开销。如果你觉得C/C++插件不够好用，我们推荐使用clangd作为C/C++的语言服务器以提供上述的功能，事实上CLion使用的语言服务器之一就是clangd。

## 还可以做到更多