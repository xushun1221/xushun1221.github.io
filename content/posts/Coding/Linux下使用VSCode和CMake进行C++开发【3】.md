---
title: Linux下使用VSCode和CMake进行C++开发【3】
date: 2022-02-14
tags: [Linux, CMake, C++]
categories: [Coding]
---

## 安装VSCode
1. 直接官网下载最新版本的deb包，地址：[https://code.visualstudio.com](https://code.visualstudio.com)；
2. 在下载deb包的目录下打开终端输入：`sudo dpkg -i xxxx.deb`，等待安装即可；
3. 安装完成后，在终端中输入：`code`打开VSCode。

-----

## 安装插件
- C/C++
- CMake
- CMake Tools

-----

## 快捷键
详情查看微软官方，https://code.visualstudio.com/shortcuts/keyboard-shortcuts-windows.pdf

-----

## 写个简单项目

### helloworld
- 在`demo2`目录下输入`code .`，通过VSCode打开目录；
- 写一个`helloworld.cpp`文件，输出`hello world`；
- Ctrl + \` 打开终端，`g++ helloworld.cpp -o helloworld`，编译文件；
- `./helloworld`可执行文件可以正确运行。

### 稍微复杂一点的项目
用类写一个简单的swap功能，如下：  
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【3】/swap目录.jpg "swap目录")
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【3】/swap结果.jpg "swap结果")
![](/post_images/posts/Coding/Linux下使用VSCode和CMake进行C++开发【1】/swapcpp.jpg "swap.cpp")

## 参考资料
1. https://www.bilibili.com/video/BV1fy4y1b7TC