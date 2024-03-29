---
title: 【MySQL】14 - MyISAM主键和二级索引树
date: 2022-11-06
tags: [datebase, MySQL]
categories: [DateBase]
---


MyISAM存储引擎采用B+索引树，数据和索引存储在两个文件中（`.MYD  .MYI`）。



## MyISAM 主键索引

> 和InnoDB不同的是，MyISAM主键索引树的叶子节点中存储的是主键key值和其对应行记录数据的**地址**而不是数据本身。


主键索引树示意图：

![](/post_images/posts/Database/MySQL/MyISAM主键索引树.jpg "MyISAM主键索引树")




## MyISAM 二级索引（辅助索引）

> 和InnoDB不同的是，MyISAM二级索引树的叶子节点中存储的是普通索引key值和其对应**行记录数据的地址**，而不是对应的主键值。



主键索引树示意图：

![](/post_images/posts/Database/MySQL/MyISAM二级索引树.jpg "MyISAM二级索引树")



## 区别？

在MyISAM中，主键索引树和二级索引树，没有区别（除了主键索引不能重复，二级索引可以重复）。

既然主键索引树和二级索引树的叶子节点都指向对应行记录数据，那么也就不存在什么回表问题。



## 聚集、非聚集索引

对于和数据存放在一起的索引，叫做**聚集（簇）索引**（InnoDB中数据存放在主键索引树中）。

而和数据分开存放的索引，叫做**非聚集（簇）索引**（MyISAM中数据存放在单独的文件中）。