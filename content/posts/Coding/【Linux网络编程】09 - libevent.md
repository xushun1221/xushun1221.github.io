---
title: 【Linux网络编程】09 - libevent
date: 2022-06-19
tags: [Linux, network]
categories: [Coding]
---

libevent库是开源的、精简的、跨平台的、专注于网络通信的库。


## 安装libevent
这里使用libevent-2.1.8版本。

libevent官方网站 [libevent.org](https://libevent.org)

解压安装过程：  
```console
# 解压
xushun@xushun-virtual-machine:~$ tar zxvf Downloads/libevent-2.1.8-stable.tar.gz
xushun@xushun-virtual-machine:~$ cd libevent-2.1.8-stable/

# 检查安装环境 生成makefile
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ ./configure 
# 生成.o和可执行文件
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ make
# 安装 将必要的资源拷贝置系统指定目录
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ sudo make install

# sample目录下有编译好的程序可以用于测试
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable$ cd sample/
# 启动hello-world服务器
xushun@xushun-virtual-machine:~/libevent-2.1.8-stable/sample$ ./hello-world

# 使用另一个终端测试
xushun@xushun-virtual-machine:~$ nc 127.1 9995
Hello, World!

# 安装完成
```

`libevent.so libevent.a`在`/usr/local/lib`目录下  
相关头文件在`/usr/local/include`目录下

## libevent的特点
- 事件驱动，高性能
- 轻量级，专注于网络
- 跨平台，支持Windows，Linux，MacOS等
- 支持多种IO多路复用技术，epoll，pool，dev/poll，select，kqueue等
- 支持IO，定时器，信号等事件

libevent是基于**事件**的**异步**通信模型。（异步通信主要依赖于**注册-回调**机制）

## Libevent 框架
使用libevent需要搭建起Libevent的框架。

1. 创建`event_base`（libevent中`event_base`是万物起源）
2. 创建事件`event`或`bufferevent`
3. 将事件添加到`event_base`
4. 启动循环监听
5. 释放`event_base`
