---
title: 【Linux系统编程】07 - 文件系统
date: 2022-05-18
tags: [Linux]
categories: [Coding]
---

## Inode和dentry
Inode的本质是一个结构体，存储文件的属性信息。如：权限、类型、大小、时间、用户、盘块位置等等。也叫作文件属性管理结构，大多数的Inode都存储在磁盘上。少量常用、近期使用的Inode会被缓存到内存中。

dentry（directory entry），目录项，本质也是一个结构体，重要的成员变量有两个（文件名，指向Inode的指针，...），目录项所对应的文件存放在磁盘中。

dentry和Inode的关系可以参考下图：  
![](/post_images/posts/Coding/【Linux系统编程】07/dentry和Inode.jpg "dentry和Inode")

- 我们的数据存储在磁盘上的某个区域内，但是我们需要一种结构来帮助我们找到这些数据存放的位置，这个结构就是Inode；
- 同时我们也要通过文件名来找到这个Inode，使用dentry目录项结构来封装文件名和Inode编号，通过Inode编号我们就能找到Inode结构体从而找到文件数据；
- 一个文件的主要就是由目录项dentry和Inode组成；
- 操作系统对文件创建硬链接的原理是，各个硬链接都有相同的Inode号，只是dentry不同，删除一个硬链接实际上就是去掉一个dentry。

## stat
