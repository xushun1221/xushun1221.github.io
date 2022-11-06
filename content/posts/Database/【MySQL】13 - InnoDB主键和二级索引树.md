---
title: 【MySQL】13 - InnoDB主键和二级索引树
date: 2022-11-05
tags: [datebase, MySQL]
categories: [DateBase]
---

InnoDB存储引擎采用B+索引树，数据和索引存储在同一个文件中（`.ibd`）。

## InnoDB 主键索引

InnoDB的主键索引树，非叶子节点存储索引key，叶子节点存储索引key和对应的数据。

做等值查询时，从根节点向下查询；  
做整表搜索或范围查询时，直接从叶子节点链表进行遍历，无需遍历整个B+树结构。


主键索引树示意图：

![](/post_images/posts/Database/MySQL/InnoDB主键索引树.jpg "InnoDB主键索引树")


### 分析sql

示例：  
```sql
mysql> DROP INDEX nameidx On student;
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> EXPLAIN SELECT * FROM student WHERE uid=5;
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type  | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | const | PRIMARY       | PRIMARY | 4       | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+-------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.01 sec)

mysql> EXPLAIN SELECT * FROM student WHERE uid<5;
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | student | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |    4 |   100.00 | Using where |
+----+-------------+---------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT * FROM student WHERE name='chenwei';
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | student | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    5 |    20.00 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
`type`字段的`const`、`range`、`ALL`分别对应等值查询、范围查询和整表扫描。






## InnoDB 二级索引（辅助索引）

InnoDB的普通索引（二级索引），非叶子节点存储普通索引key（不同于主键，主键不允许重复，普通索引可以重复），叶子节点存储普通索引key和其对应的记录行的主键值。


使用带二级索引的字段进行过滤时，如果查询的字段仅为主键或二级索引字段，则直接在该二级索引树上搜索即可。

如果查询的字段不是主键或二级索引对应的字段，则需要**回表**，回表是说，在二级索引树上搜索得到主键，再用主键在主键索引树上搜索其对应的记录行中的其他字段值。



二级索引树示意图：

![](/post_images/posts/Database/MySQL/InnoDB二级索引树.jpg "InnoDB二级索引树")



### 分析sql

给`student`表的`name`字段加上索引：`CREATE INDEX nameidx ON student(name);`

分析几个sql：  
1. `SELECT name FROM student WHERE name='chenwei';`
    `name`字段有索引，查询的字段也是`name`，直接在二级索引树上完成搜索了
2. `SELECT uid,name FROM student WHERE name='chenwei';`
    `uid`是主键，存放在二级索引树的叶子节点中，所以也可以直接在二级索引树上完成搜索
3. `SELECT * FROM student WHERE name='chenwei';`
   1. 在二级索引树上搜索`chenwei`对应的主键`uid`
   2. 再用该主键在主键索引树上搜索`uid`那一行的记录数据

```sql
mysql> EXPLAIN SELECT name FROM student WHERE name='chenwei';
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | student | NULL       | ref  | nameidx       | nameidx | 152     | const |    1 |   100.00 | Using index |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT uid,name FROM student WHERE name='chenwei';
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | student | NULL       | ref  | nameidx       | nameidx | 152     | const |    1 |   100.00 | Using index |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN SELECT * FROM student WHERE name='chenwei';
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type | possible_keys | key     | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | student | NULL       | ref  | nameidx       | nameidx | 152     | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+------+---------------+---------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

> 涉及到查询时，尽量避免使用`SELECT * ...`，最好需要哪些字段就写哪些。


## 多列索引（联合索引）

`SELECT * FROM student WHERE age=20 ORDER BY name;`，如果要进行这样的查询，应该怎么做效率最高呢？

对同一个表进行搜索，我们不能同时使用两个索引，而应该使用**多列索引（联合索引）**。

`age`有索引，`name`无索引，出现了`Using filesort`，效率很低：  
```sql
----+-------------+---------+------------+------+---------------+--------+---------+-------+------+----------+---------------------------------------+
| id | select_type | table   | partitions | type | possible_keys | key    | key_len | ref   | rows | filtered | Extra                                 |
+----+-------------+---------+------------+------+---------------+--------+---------+-------+------+----------+---------------------------------------+
|  1 | SIMPLE      | student | NULL       | ref  | ageidx        | ageidx | 1       | const |    2 |   100.00 | Using index condition; Using filesort |
+----+-------------+---------+------------+------+---------------+--------+---------+-------+------+----------+---------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

添加联合索引`(age,name)`（先按age排序，再按name排序），构建联合索引的时候，已经按照`age,name`的顺序进行了排序，就不需要再对`name`进行排序了，自然也就没有`Using filesort`，效率提高  
```sql
mysql> CREATE INDEX age_name_idx ON student(age,name);
Query OK, 0 rows affected (0.01 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> EXPLAIN SELECT * FROM student WHERE age=20 ORDER BY name;
+----+-------------+---------+------------+------+---------------+--------------+---------+-------+------+----------+-----------------------+
| id | select_type | table   | partitions | type | possible_keys | key          | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+---------+------------+------+---------------+--------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | student | NULL       | ref  | age_name_idx  | age_name_idx | 1       | const |    2 |   100.00 | Using index condition |
+----+-------------+---------+------------+------+---------------+--------------+---------+-------+------+----------+-----------------------+
1 row in set, 1 warning (0.00 sec)
```
当然了，这个sql也需要回表，因为这个二级索引树上只有`age`、`name`、`uid`三列。


> 注意，使用多列索引，必须使用多列索引的第一个索引，因为多列索引的第一个索引列是最优先进行排序的。
