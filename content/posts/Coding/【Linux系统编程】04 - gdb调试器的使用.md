---
title: 【Linux系统编程】04 - gdb调试器的使用
date: 2022-05-13
tags: [Linux, gdb]
categories: [Coding]
---
 
参考这里一起看[gdb介绍](https://xushun1221.github.io/2022/linux%E4%B8%8B%E4%BD%BF%E7%94%A8vscode%E5%92%8Ccmake%E8%BF%9B%E8%A1%8Cc-%E5%BC%80%E5%8F%912/)

 ## gdb使用前提
 使用gdb调试器进行调试的前提条件，是对文件进行编译时添加`-g`选项，为可执行文件添加调试信息，这样会使编译后的文件变大一些。

 使用gdb进行调试，主要是为了找出程序的逻辑错误，而程序的语法错误在gcc编译阶段就会被发现。

 ## gdb的使用
 这里我们写一个简单的选择排序的c程序`test.c`用来测试。  
1. 首先进行编译，`gcc test.c -g -o gdbtest`；
2. `gdb gdbtest`，使用gdb工具开始调试，此时进入gdb命令模式，可以输入命令操作。

## gdb的基础命令
- `list 或 l`，显示源码，`l 1`，从第一行开始显示10行，按一次`l`向下显示10行；
- `break/b n`，在第n行设置断点；
- `run/r`，开始运行到断点位置，断点位置未执行；
- `next/n`，运行到下一行，不会进入函数；（如果是系统函数不能`s`，只能`n`）
- `step/s`，单步执行，会进入函数；
- `print/p var`，查看`var`变量的值；
- `contitnue`，运行到下一个断点位置，如果没有断点了，就运行到结束；
- `quit`，退出gdb。

## gdb的其他操作
- `run`，用`run`命令执行程序，如果程序出错，会停在出错位置；
- `start`，从第一行开始执行；
- `finish`，结束当前函数调用；
- `set args`，设置main的命令行参数，在运行之前设置；
- `run str1 str2 ...`，设置命令行参数；
- `info b`，查看断点信息表；
- `b 20 if i=5`，设置条件断点；
- `ptype var`，查看变量类型；
- `bt`，列出当前程序正存活着的栈帧；
- `frame`，根据栈帧编号，切换栈帧；
- `display var`，设置跟踪变量；
- `undisplay var的编号`，取消设置跟踪变量，使用跟踪变量的编号。