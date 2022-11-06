---
title: 【MySQL】15 - 哈希索引
date: 2022-11-06
tags: [datebase, MySQL]
categories: [DateBase]
---


## 哈希索引

用于搜索的数据结构常见的就是，平衡树和哈希表。存储引擎也是，InnoDB和MyISAM引擎支持B+树索引，而MEMORY引擎（基于内存）可以支持哈希索引。


### 哈希索引的特性

常用的哈希表结构是，链式哈希表。

1. 哈希表中的元素是没有顺序的，它只能支持等值搜索，对于范围搜索、前缀搜索、ORDER BY排序等，无法支持。
2. 如果哈希表中，桶里的一个节点代表一次磁盘IO的话，做整表扫描、范围搜索的情况下，磁盘IO的次数非常多，效率低下。所以哈希索引，只能用于内存中。


### 查看索引类型

创建哈希索引：  
```sql
mysql> CREATE INDEX nameidx ON student(name) USING HASH;
Query OK, 0 rows affected (0.00 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

可以看到`name`后面标注了`USING HASH`，但是mysql server是否真的使用了哈希索引，这里的显示并不准确  
```sql
mysql> SHOW CREATE TABLE student\G
*************************** 1. row ***************************
       Table: student
Create Table: CREATE TABLE `student` (
  `uid` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(50) NOT NULL,
  `age` tinyint(3) unsigned NOT NULL,
  `sex` enum('m','w') NOT NULL,
  PRIMARY KEY (`uid`),
  KEY `nameidx` (`name`) USING HASH
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
```

这样查询才是准确的，可以看到`Index_type`项为`BTREE`，所以其实没有用哈希索引。  
```sql
mysql> SHOW INDEXES FROM student;
+---------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| Table   | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment |
+---------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
| student |          0 | PRIMARY  |            1 | uid         | A         |           5 |     NULL | NULL   |      | BTREE      |         |               |
| student |          1 | nameidx  |            1 | name        | A         |           5 |     NULL | NULL   |      | BTREE      |         |               |
+---------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+
2 rows in set (0.00 sec)
```







