---
title: 【MySQL】29 - SQL的完整处理流程
date: 2022-11-14
tags: [datebase, MySQL]
categories: [DateBase]
---

![](/post_images/posts/Database/MySQL/SQL处理流程.jpg "SQL处理流程")

- 连接器：管理连接，权限验证
- 解析器：词法以及语法分析
- 优化器：生成执行计划，选择合适索引
- 执行器：调用存储引擎接口，进行读写操作
- 存储引擎：提供读写接口，花费磁盘IO进行读写，构建B+树，管理事务日志
- 磁盘数据：表结构、表数据、表索引

