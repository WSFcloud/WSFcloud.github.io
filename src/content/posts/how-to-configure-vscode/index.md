---
title: Visual Studio Code配置指南
published: 2024-10-18
description: ''
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
事实上，VS Code是一个编辑器而非IDE，只有编辑器是无法像Visual Studio等IDE一样编译C/C++代码的，因此我们需要安装一个编译器。在Windows上就我们推荐安装MinGW-W64（注意是MinGW-W64不是MinGW）作为编译器。MinGW是GNU工具（包括编译器GCC、GNU binutils和调试器GDB等）在Windows上的一个移植，但是它只支持32位，而MinGW-W64是MinGW的一个分支，它提供了64位支持。

MinGW-W64有许多发行版本如mingw-builds、MSYS2、LLVM-MinGW等，这里我们选择mingw-builds。在[这里](https://github.com/niXman/mingw-builds-binaries/releases)选择一个版本下载，本文选择x86_64-14.2.0-release-posix-seh-ucrt-rt_v12-rev0.7z作为安装版本。

下载完后得到一个拓展名为.7z的压缩包，解压后得到的就是MinGW-W64本体，把它放在一个你能找到的位置比如C盘根目录下面（不管放到哪里，路径不要含有中文防止出现奇怪的错误）。接下来将它添加到Path，右键“此电脑”-属性-高级系统设置-环境变量，选择系统变量下的Path项点击编辑-新建，将刚才解压得到的MinGW文件夹里的bin文件夹路径写进去（如：C:\mingw64\bin）保存后MinGW就被添加到了Path里面。

打开Windows终端（或者你也可以在VS Code里点击终端-新建终端）输入gcc -v，回车后你会看见关于MinGW-W64的版本信息，如果提示“ 'gcc'不是内部或外部命令，也不是可运行的程序或批处理文件。”说明没有成功添加到Path，请检查是不是出现了拼写错误或路径错误。

## 配置Visual Studio Code
